<div align="center">

# 🐴 Automações Axial Academy

**Dois sistemas de automação construídos para a Axial Academy: atendimento via IA no WhatsApp e distribuição de conteúdo para grupos**

![Status](https://img.shields.io/badge/status-em%20produção-brightgreen)
![Platform](https://img.shields.io/badge/platform-Make.com-6D28D9)
![Channel](https://img.shields.io/badge/canal-WhatsApp-25D366)

</div>
---

## 👤 Sobre este projeto

Desenvolvido por Julia Bezerra como projeto freelance para a Axial Academy.

<div align="center">

*Documentação técnica · Última atualização: setembro/2026*

</div>

Este repositório documenta dois fluxos de automação distintos, desenvolvidos no mesmo contrato para a Axial Academy:

1. **[Eduardo](#-projeto-1--eduardo-agente-de-atendimento)** — assistente virtual de IA para triagem e admissões do Equine Spine Focus.
2. **[Automação de Conteúdo](#-projeto-2--automação-de-conteúdo-instagramyoutube--grupos-whatsapp)** — captação e distribuição controlada de posts/vídeos para grupos de WhatsApp.

Ambos rodam na mesma conta do Make e compartilham o mesmo pool de créditos mensais

---
### ✨ Destaques técnicos

- 🤖 **Bot de WhatsApp com IA (Gemini)** rodando em paralelo a um fluxo 100% determinístico para alunos já matriculados — sem misturar os dois públicos.
- 🔒 **Camadas de segurança no atendimento**: rede de segurança por palavra-chave (transbordo humano mesmo se a IA errar), proibição de diagnóstico clínico e bloqueio de imagens/áudios chegando à IA.
- 📲 **Sistema de distribuição de conteúdo** com fila unificada (captação automática + manual), cadência anti-spam (1 disparo/4 dias) e segmentação por grupo.
- ⚙️ **Dois sistemas independentes coexistindo na mesma conta Make**, compartilhando pool de créditos e limite de cenários ativos — com documentação separada para não confundir escopos.

## 📑 Sumário

**Projeto 1 — Eduardo**
- [Visão geral do fluxo](#11-visão-geral-do-fluxo)
- [Data Store](#12-data-store-único-por-limite-do-plano)
- [Mapa de módulos](#13-mapa-de-módulos-principais)
- [Prompt do Gemini](#14-prompt-do-gemini-módulo-62)
- [Regras de segurança](#15-regras-críticas-de-segurança)
- [Pendências](#16-pendências--itens-em-aberto)
- [Checklist de testes](#17-checklist-de-testes-de-ponta-a-ponta)

**Projeto 2 — Automação de Conteúdo**
- [Visão geral](#21-visão-geral)
- [Stack tecnológica](#22-stack-tecnológica)
- [Arquitetura](#23-arquitetura)
- [Estrutura da planilha](#24-estrutura-da-planilha-google-sheets)
- [Regras de negócio](#25-regras-de-negócio)
- [Cenários no Make](#26-cenários-no-make)
- [Troubleshooting](#27-troubleshooting-comum)

**Compartilhado**
- [Recursos compartilhados](#-recursos-compartilhados)
- [Custos mensais totais](#-custos-mensais-totais)
- [Acessos e propriedade](#-acessos-e-propriedade)

---

## 🤖 Projeto 1 — Eduardo (Agente de Atendimento)

Assistente virtual de WhatsApp para triagem e admissões do curso **Equine Spine Focus**, integrado à **WhatsApp Cloud API** (via Coexistência) e ao **Google Gemini** para geração de respostas.

### 1.1 Visão geral do fluxo

<img width="1754" height="743" alt="image" src="https://github.com/user-attachments/assets/0c73e2bf-9e2e-438b-8d5c-ef4223496977" />


**Fluxo "não é aluno" (com IA):**

1. Verifica cache do manual (Data Store) → busca no Google Docs se não existir.
2. Verifica histórico da conversa daquele telefone (Data Store).
3. Verifica se o telefone já está em `status: transbordo_humano` → se sim, **não responde nada**.
4. Chama o Gemini com: roteiro + histórico + mensagem nova.
5. Faz parse do JSON de resposta da IA.
6. Busca os links reais (Data Store) e substitui os marcadores `Link 01`–`Link 06` no texto.
7. Router de transbordo: decisão da IA **OU** palavra-chave (`humano`, `atendente`, `pessoa de verdade`, `falar com alguém`) na mensagem crua do cliente.

**Fluxo "é aluno matriculado" (sem IA, 100% determinístico):**

1. **Status vazio** (1ª mensagem) → envia Mensagem 1 fixa → grava `status: aguardando_resposta_aluno`.
2. **Status = aguardando_resposta_aluno** (2ª mensagem) → envia Mensagem 2 (transbordo) → grava `status: transbordo_humano`.
3. **Status = transbordo_humano** (3ª mensagem em diante) → **não responde nada**.

### 1.2 Data Store 

> Todos os "assuntos" convivem numa única estrutura de 10 colunas, diferenciados pela **key** de cada registro.

| Campo | Tipo | Usado por |
|---|---|---|
| `link01`–`link06` | Text | registro `key: "atual"` |
| `conteudo` | Text | registro `key: "roteiro_whatsapp"` |
| `atualizado_em` | Date | registro `key: "roteiro_whatsapp"` |
| `mensagens` | Text | registros `key: <telefone>` |
| `status` | Text | registros `key: <telefone>` |

| Key | Conteúdo | Como é criada |
|---|---|---|
| `roteiro_whatsapp` | Cache do manual (Google Docs) | Automática |
| `atual` | Os 6 links reais | **Manual** — editar direto na aba Records |
| `<telefone>` | Histórico + status da conversa | Automática, na 1ª mensagem |

> ⚠️ **Sem TTL automático.** Editou o Google Doc do manual? Apague o registro `roteiro_whatsapp` (Data Stores → Data Store → Records) para a mudança valer.

### 1.3 Mapa de módulos principais

<details>
<summary>Clique para expandir a tabela completa de módulos</summary>
| ID | Módulo | Função |
|---|---|---|
| 88 | Watch Events | Trigger — recebe eventos da WhatsApp Cloud API (mensagens, status, e potencialmente `smb_message_echoes`) |
| 149/150/152 | ExistRecord + IfElse + GetRecord | Lê o `status` daquele telefone, protegido contra telefone novo |
| 157 | Router | Bloqueia **qualquer** mensagem (texto, áudio, imagem) se `status = transbordo_humano`, antes mesmo de checar o tipo |
| 10 / 11 | Router + Send Message | Separa mensagem de texto vs. outros tipos |
| 3 | Google Sheets – Search Rows | Verifica se o telefone é aluno matriculado (**coluna H**, formato só-dígitos) |
| 42 | Router | Separa fluxo "é aluno" vs. "não é aluno" |
| 20/25/18/19/49/63 | ExistRecord + IfElse + GetRecord/AddRecord | Cache do roteiro (Google Docs) |
| 103/104/85 | ExistRecord + IfElse + GetRecord | Leitura protegida do histórico |
| 91/92 | Router | Checa `status = transbordo_humano` antes de chamar a IA (camada redundante com o 157, mantida por segurança) |
| 134 | Gemini AI – Extract structured data | Geração da resposta com Response Schema nativo |
| 142 | Retry (error handler do 134) | Reprocessa automaticamente em caso de instabilidade momentânea do Gemini |
| 80 | GetRecord | Busca os 6 links reais + `pdf_media_id` |
| 73 | Set Variable | Substitui `Link 0X` pelos links reais (`resposta_final`) |
| 179 | Router | Decide entre texto normal (90), template com botão (ver 1.7) e fallback |
| 90 | Send a Message | Envio da resposta normal |
| 168 | Router (rotas não-exclusivas) | Dispara em paralelo: PDF (se `enviar_pdf`) e "à disposição" (se `etapa` contém "passo2") |
| 127 | Send a Message (Document) | Envia o PDF usando `pdf_media_id` |
| 170 | Send a Message | "Qualquer dúvida, estou à disposição." |
| 74/75/76/89 | Router + AddRecord + Send Message | Decide e executa transbordo vs. envio normal |
| 113/114/95 | ExistRecord + IfElse + GetRecord | Leitura protegida do status do aluno |
| 97/44/101 | Router + Send Message + AddRecord | Reconhecimento do aluno + transbordo imediato (mensagem única) |
 
</details>


### 1.4 Prompt do Gemini (módulo 62)

**Migração importante:** o projeto começou com o módulo `Google Gemini AI – Simple Text Prompt` (pedindo à IA que respondesse em JSON via instrução de texto) mais um módulo de limpeza (`replace()` para remover blocos ` ```json `) e um `Parse JSON` separado. Essa abordagem quebrava com frequência (a IA ocasionalmente embrulhava a resposta em markdown, principalmente com o prompt mais longo).
 
**Hoje o módulo 134 usa a ação `Extract structured data`**, que aceita um **Response Schema** nativo — a Meta/Google garante a validade do JSON na origem, sem necessidade de limpeza nem parse manual. Os módulos antigos (Gemini simples + limpeza + Parse JSON) foram removidos.
 
**Campos do Response Schema (Required marcado onde a IA sempre decide algo; não-obrigatório onde o valor pode legitimamente não existir ainda):**
 
| Campo | Tipo | Required | Motivo |
|---|---|---|---|
| `resposta` | String | Sim | sempre gerado |
| `transbordo` | Boolean | Sim | decisão sempre tomada |
| `perfil` | String | Não | só existe após o Passo 1 |
| `elegivel` | Boolean | Não | só existe para estudantes, após as 2 perguntas |
| `etapa` | String | Sim | usada em lógica crítica downstream |
| `enviar_pdf` | Boolean | Sim | decisão sempre tomada (default false) |
| `responder` | Boolean | Sim | decide se o texto da IA deve ser enviado ou substituído por "à disposição" |
| `botoes` | String | Sim | qual template de botão usar (`"nenhum"` na maioria dos casos) |
| `primeiro_nome` | String | Sim | usado para variável de template (separado de `resposta`, que não é usada quando a mensagem sai via template) |
 
**Modelo em uso:** `gemini-3.1-flash-lite` 
 
**Padrão de nomenclatura de `etapa`** (usado por filtros em vários pontos do fluxo — precisa ser respeitado ao editar o prompt):
- Primeira mensagem / menu ainda não respondido → não contém "passo2/3/4" (ex: `"cenario_b_passo1"`).
- Resposta ao menu 1/2/3 → contém `"passo2"`.
- Resposta ao follow-up ("o que achou da proposta?") → contém `"passo3"`.
- Continuação depois disso (matrícula, motivo, etc.) → contém `"passo4"`.

Ordem do prompt: instruções e regras → formato dos marcadores de link → formato JSON acima → roteiro (`{{63.conteudo}}`) → histórico (`{{ifempty(85.mensagens; ...)}}`) → nome do cliente (`{{88.contacts[].profile.name}}`) → mensagem nova (`{{88.messages[].text.body}}`).


### 1.5 Regras críticas de segurança

- **Fidelidade textual** — frases entre aspas no manual são fixas; a IA nunca parafraseia.
- **Proibição de diagnóstico** — nunca avalia clinicamente nem recomenda tratamento.
- **Imagens/áudios nunca chegam à IA** — recebem resposta fixa pedindo texto.
- **Links nunca são gerados pela IA** — só marcadores; URL real inserida depois, de forma determinística.
- **Nunca pular etapa não respondida** — só avança para "Estudante Elegível/Não Elegível" se as duas perguntas anteriores tiverem resposta real e visível no histórico.

### 1.6 Melhorias futuras opicionais

- [ ] Implementar expiração automática do cache do roteiro.
- [ ] Avaliar tabela de log/auditoria separada.
- [ ] Implementar botões nos links das mensagens

### 1.7 Checklist de testes de ponta a ponta

1. Cliente novo, não aluno, frase-gatilho do Cenário A → recebe menu 1/2/3.
2. Continuidade de histórico (IA lembra do contexto anterior).
3. Pedido explícito de "falar com humano" → transbordo, bot silencia depois.
4. Aluno matriculado, 1ª mensagem → Mensagem 1 fixa.
5. Aluno matriculado, 2ª mensagem → Mensagem 2 (transbordo).
6. Aluno matriculado, 3ª mensagem → silêncio.
7. Mensagem não-textual → resposta fixa, sem passar pela IA.

---

## 📲 Projeto 2 — Automação de Conteúdo (Instagram/YouTube → Grupos WhatsApp)

Sistema que capta publicações de Instagram e YouTube e as distribui de forma controlada e segmentada em grupos de WhatsApp, respeitando regras de cadência e segurança anti-banimento.

### 2.1 Visão geral

| | |
|---|---|
| **Fontes** | 3 contas de Instagram + 1 canal de YouTube |
| **Destino** | 3 grupos de WhatsApp |
| **Volume** | Até 20 conteúdos/mês |
| **Cadência de envio** | 1 disparo a cada 4 dias (96h) |
| **Segmentação** | Por grupo, definida manualmente por conteúdo |

### 2.2 Stack tecnológica

| Ferramenta | Função |
|---|---|
| **Make** | Motor de automação — orquestra captação, fila e disparo |
| **Z-API** | Ponte de conexão com o WhatsApp (número dedicado, não-oficial) |
| **Instagram for business** | Gera feeds RSS das contas de Instagram |
| **Google Sheets** | Banco de dados / fila de controle de todo o conteúdo |
| **YouTube RSS nativo** | Feed público, sem necessidade de ferramenta extra |

### 2.3 Arquitetura

```text
┌──────────────┐     ┌──────────────┐
│ YouTube (RSS)│     │ Instagram x3 │
│    nativo     │     │ │
└───────┬───────┘     └───────┬──────┘
        │                     │
        ▼                     ▼
   4 Cenários de Captação (Make)
   (1 por fonte — Make só aceita
    1 gatilho por cenário)
        │
        ▼
┌─────────────────────┐
│   Google Sheets       │
│   (Fila de Conteúdo)  │
└──────────┬────────────┘
           ▼
   Cenário de Disparo 
   Roda a cada 4 dias 
           │
           ▼
   Google Sheets → Search Rows
   (item mais antigo, status "Pendente")
           │
           ▼
     Compose a String
     (monta a mensagem)
           │
           ▼
        Router
   ┌───────┼───────┐
 Grupo1  Grupo2  Grupo3
 (Filter: coluna = "SIM")
   │       │       │
 Z-API   Z-API   Z-API
   │       │       │
 Sleep   Sleep   Sleep
  15s     15s     15s
   │       │       │
Update  Update  Update
 Row     Row     Row
```

### 2.4 Estrutura da planilha (Google Sheets)

| Coluna | Descrição |
|---|---|
| Origem | "YouTube" ou "Instagram" |
| Link | URL do conteúdo |
| Título | Título do vídeo/post |
| Data de Captação | Formato `YYYY-MM-DD` (importante para ordenação correta da fila) |
| Grupo 1 / 2 / 3 | "SIM" ou "NÃO" — define a segmentação |
| Status | "Pendente" ou "Enviado" |
| Data de Envio | Preenchida automaticamente após o disparo |

### 2.5 Regras de negócio

1. **Fila unificada** — conteúdos captados automaticamente e inseridos manualmente entram na mesma fila.
2. **Cadência controlada** — 1 disparo a cada 4 dias, sempre o item mais antigo pendente.
3. **Segmentação por grupo** — cada conteúdo só vai aos grupos marcados como "SIM".
4. **Delay de segurança** — 15 segundos entre o envio a cada grupo.
5. **Número dedicado** — roda em um número de WhatsApp separado dos números pessoal e comercial.

### 2.6 Cenários no Make

| Cenário | Gatilho | Intervalo |
|---|---|---|
| Captação - YouTube | Watch RSS Feed Items | 15-30 min |
| Captação - Instagram 1 | Watch RSS Feed Items | 15-30 min |
| Captação - Instagram 2 | Watch RSS Feed Items | 15-30 min |
| Captação - Instagram 3 | Watch RSS Feed Items | 15-30 min |
| Disparo | Google Sheets Search Rows | 5760 min (4 dias) |

> ⚠️ São 5 cenários ativos simultaneamente — excede o limite de 2 do plano gratuito do Make, exigindo o plano Core (pago). **Somado ao Projeto 1, a conta já roda 6+ cenários ativos ao todo.**

**Adicionando conteúdo antigo manualmente:** abra a planilha → nova linha → preencha Origem/Link/Título → marque os grupos (SIM/NÃO) → Status = "Pendente". Entra na fila normalmente, ordenado pela Data de Captação.

### 2.7 Troubleshooting comum

| Problema | Causa provável |
|---|---|
| Mesmo item capturado repetidamente | Erro em módulo downstream impedindo o "checkpoint" do RSS de avançar |
| Mensagem não chega no grupo | ID do grupo incorreto ou instância Z-API desconectada |
| Fila em ordem errada | Data de Captação fora do formato `YYYY-MM-DD` |
| Erro 415 na Z-API | Faltando header `Content-Type: application/json` |

---

## 🔗 Recursos compartilhados

Os dois projetos rodam na **mesma conta do Make**, o que significa:

- **Créditos somados** — o consumo do Projeto 1 (Eduardo, ~15-20 créditos por mensagem de cliente) e do Projeto 2 (5 cenários poll/schedule + disparos) contam para o **mesmo pool mensal** de créditos do plano. Ao dimensionar o plano, considerar o volume dos dois juntos, não separadamente.
- **Contagem de cenários ativos** — irrelevante no plano Core (ilimitado), mas relevante caso a conta algum dia volte para o Free (limite de 2 no total, somando os dois projetos).
- **Contas Google** — Projeto 1 usa Google Docs + Sheets; Projeto 2 usa Google Sheets. Se estiverem no mesmo Google Workspace da empresa, ambos compartilham a mesma cota de API do Google (raramente um problema no volume atual).
- **Sem sobreposição de Data Store** — o Projeto 2 não usa Data Store do Make, então não disputa o limite de "1 Data Store por conta" do Projeto 1.

## 💰 Custos mensais totais

| Item | Custo estimado |
|---|---|
| Make (plano Core, compartilhado pelos 2 projetos) | R$ 50 |
| Z-API (Projeto 2) | R$ 99 |
| Gemini API (Projeto 1) | Gratuito no volume atual (tier free) |
| WhatsApp Cloud API (Projeto 1) | Conversas gratuitas até a cota mensal; cobrança por conversa acima disso |
| **Total estimado** | **R$ 150 /mês** |

## 🔑 Acessos e propriedade

Todas as contas (Make, Z-API, RSS.app, Google, Meta Business) devem estar cadastradas em nome/e-mail da **Axial Academy**, não do desenvolvedor — garantindo continuidade caso haja troca de responsável técnico no futuro.

---

<div align="center">

*Documentação mantida pela equipe Axial Academy · Última atualização: setembro/2026*

</div>
