<div align="center">

# 🐴 Automações Axial Academy

**Dois sistemas de automação no Make para a Axial Academy: um agente de IA que atende no WhatsApp e um sistema que distribui conteúdo do Instagram e do YouTube para grupos**

![Status](https://img.shields.io/badge/status-concluído%20·%20em%20evolução-brightgreen)
![Platform](https://img.shields.io/badge/platform-Make.com-6D28D9)
![AI](https://img.shields.io/badge/IA-Claude%20(Anthropic)-D97757)
![Channel](https://img.shields.io/badge/canal-WhatsApp-25D366)

*Documentação técnica · Última atualização: novembro/2026*

</div>

---

## 👤 Sobre este projeto

Projeto freelance que desenvolvi, Julia Bezerra, para a **Axial Academy** (educação em ortopedia e biomecânica equina). Os dois sistemas fizeram parte do mesmo contrato, por isso estão neste repositório:

1. **[Eduardo](#-projeto-1-eduardo-agente-de-atendimento-com-ia)**: um agente de IA no WhatsApp que faz a triagem e o atendimento de interessados no curso **Equine Spine Focus**. Ele trabalha com um cenário de **follow-up automático**, que volta a falar com quem parou de responder.
2. **[Automação de Conteúdo](#-projeto-2-automação-de-conteúdo-instagramyoutube--grupos-whatsapp)**: capta os posts do Instagram e os vídeos do YouTube e os envia para os grupos de WhatsApp da comunidade, com uma fila controlada.

São **5 cenários** que rodam na mesma conta do Make e usam o mesmo pool de créditos.

> 🚧 **Status:** os dois projetos estão **concluídos e em produção** nesta versão. Mesmo assim, há **novas atualizações e expansões a caminho** (veja [Próximos passos](#-próximos-passos)).

---

## ✨ Destaques técnicos

- 🤖 **Agente de IA com Claude (Anthropic)** e **Structured Outputs** (JSON Schema nativo). A resposta já chega em JSON válido, então não é preciso limpar texto nem tratar markdown.
- 🛟 **Tolerância a falhas**: se a API da IA ou o parse do JSON falharem, um *error handler* devolve uma resposta segura e passa o atendimento para um humano. O cliente nunca fica sem resposta.
- 🔒 **Várias camadas de segurança**: lista de números bloqueados, transbordo por palavra-chave (que funciona mesmo se a IA errar), silêncio total depois do transbordo, proibição de diagnóstico clínico e links inseridos de forma determinística (a IA nunca escreve uma URL).
- 🔁 **Follow-up inteligente**: o tempo de espera e a mensagem mudam conforme o **perfil do lead** (veterinário, estudante ou treinador). Só envia em horário comercial (7h–21h).
- 🧭 **Fluxos separados por público**: interessados passam pela IA e alunos já matriculados seguem um fluxo determinístico, direto para a equipe.
- 📲 **Distribuição de conteúdo** com fila única no Google Sheets, cadência anti-spam, segmentação por grupo e tratamento por tipo de mídia (vídeo, imagem ou carrossel).
- 🔗 **Integração entre as fontes**: quando um post do Instagram anuncia um vídeo novo do YouTube, ele não vira um disparo duplicado. A legenda dele é **incorporada à mensagem do vídeo**.
- ⚙️ **Execução sequencial** no cenário do agente, para que duas mensagens seguidas do mesmo cliente não sobrescrevam o histórico uma da outra.

## 📑 Sumário

**Projeto 1: Eduardo**
- [Visão geral do fluxo](#11-visão-geral-do-fluxo)
- [Modelo de IA e saída estruturada](#12-modelo-de-ia-e-saída-estruturada)
- [Data Store](#13-data-store)
- [Mapa de módulos](#14-mapa-de-módulos-principais)
- [Cenário de follow-up](#15-cenário-de-follow-up)
- [Regras de segurança](#16-regras-críticas-de-segurança)
- [Checklist de testes](#17-checklist-de-testes-de-ponta-a-ponta)

**Projeto 2: Automação de Conteúdo**
- [Visão geral](#21-visão-geral)
- [Stack tecnológica](#22-stack-tecnológica)
- [Arquitetura](#23-arquitetura)
- [Estrutura da planilha](#24-estrutura-da-planilha-google-sheets)
- [Regras de negócio](#25-regras-de-negócio)
- [Cenários no Make](#26-cenários-no-make)
- [Troubleshooting](#27-troubleshooting-comum)

**Geral**
- [Recursos compartilhados](#-recursos-compartilhados)
- [Custos mensais](#-custos-mensais)
- [Próximos passos](#-próximos-passos)
- [Acessos e propriedade](#-acessos-e-propriedade)

---

## 🤖 Projeto 1: Eduardo (Agente de Atendimento com IA)

Assistente virtual de WhatsApp para triagem e admissões do curso **Equine Spine Focus**. Ele se conecta à **WhatsApp Business Cloud API** (por Coexistência, no mesmo número comercial) e usa o **Claude**, da Anthropic, para gerar as respostas a partir de um roteiro oficial guardado no Google Docs.

São dois cenários no Make:

| Cenário | Gatilho | Função |
|---|---|---|
| **Fluxo Agente WhatsApp** | Webhook da WhatsApp Cloud API (instantâneo) | Recebe as mensagens, faz a triagem e responde com IA |
| **Follow-up** | Agendado (execução periódica) | Volta a falar com leads que pararam de responder em alguma etapa |

### 1.1 Visão geral do fluxo

<img width="1619" height="574" alt="image" src="https://github.com/user-attachments/assets/e4b4cd78-78df-4866-8644-5ea4cd027bdd" />


**Filtros de entrada (antes de qualquer resposta):**

1. **Lista de números não conversáveis** (Google Sheets): se o número estiver na lista, o registro é marcado como `transbordo_humano` e o bot **não responde**.
2. **Status do contato** (Data Store): se o contato já estiver em `transbordo_humano`, o bot **não responde nada**, seja texto, áudio ou imagem.
3. **Tipo de mensagem**: áudio, imagem e outros tipos que não são texto **nunca chegam à IA**. O contato é encaminhado direto para a equipe.
4. **Aluno matriculado?** O bot procura o telefone na planilha de inscrições confirmadas e decide o caminho.

**Fluxo "não é aluno" (com IA):**

1. Lê o roteiro do cache (Data Store). Se não existir, busca no Google Docs e grava o cache.
2. Lê o histórico da conversa daquele telefone (Data Store).
3. Identifica a **origem do contato** (anúncio ou orgânico) na primeira mensagem, para que a IA escolha a abertura certa do roteiro.
4. Chama o **Claude** com o roteiro, o nome do cliente, a origem, o histórico e a mensagem nova.
5. Faz o parse do JSON estruturado. Se falhar, entra o *fallback* de transbordo seguro.
6. Busca os links reais (Data Store) e troca os marcadores `Link 01`–`Link 06` no texto.
7. **Router de transbordo**: a decisão da IA **ou** uma palavra-chave na mensagem original do cliente (`humano`, `atendente`, `pessoa de verdade`, `falar com alguém`).
8. Envia a resposta e, quando for o caso, manda **em paralelo** o PDF de investimento e a mensagem "Qualquer dúvida, estou à disposição!".
9. Atualiza o registro do lead com o histórico, o perfil, a elegibilidade e os horários de cada etapa, que alimentam o follow-up.

**Fluxo "é aluno matriculado" (sem IA, 100% determinístico):**

1. **Primeira mensagem**: o bot reconhece o aluno pelo nome e avisa que vai direcioná-lo para a equipe. Grava `status: transbordo_humano`.
2. **Mensagens seguintes**: o bot **não responde**, porque a equipe humana já assumiu o atendimento.

### 1.2 Modelo de IA e saída estruturada

**Evolução da arquitetura de IA:**

| Versão | Abordagem | Problema / Ganho |
|---|---|---|
| v1 | Gemini "Simple Text Prompt" + limpeza com `replace()` + `Parse JSON` | A IA às vezes embrulhava o JSON em markdown e o fluxo quebrava |
| v2 | Gemini "Extract structured data" com Response Schema | JSON válido na origem, sem módulos de limpeza |
| **v3 (atual)** | **Claude Sonnet 5.5 (Anthropic) + Structured Outputs (`json_schema`)** | Mais fiel ao roteiro (copia os textos fixos caractere por caractere) e com *fallback* em caso de erro |

**Configuração atual:** modelo `claude-sonnet-5-5`, *effort* `medium`, *extended thinking* desligado e `max_tokens` de 1000. O schema tem `additionalProperties: false` e todos os campos são obrigatórios.

**Campos do JSON Schema:**

| Campo | Tipo | Uso no fluxo |
|---|---|---|
| `resposta` | string | Texto enviado ao cliente (com marcadores de link) |
| `transbordo` | boolean | Encaminha para atendimento humano |
| `responder` | boolean | Envia a resposta da IA ou uma mensagem de encerramento |
| `perfil` | string \| null | `veterinario`, `estudante` ou `treinador` (define o follow-up) |
| `elegivel` | boolean \| null | Elegibilidade do estudante para o curso |
| `etapa` | string | Etapa do roteiro (usada pelos filtros que vêm depois) |
| `enviar_pdf` | boolean | Dispara o envio do PDF de investimento |
| `passo2_concluido` | boolean | Marca o fim da qualificação e inicia o contador do follow-up |
| `nome_cliente` | string | Primeiro nome, salvo para personalizar o follow-up |

**Engenharia de prompt:**
- O roteiro tem uma **"Ordem de decisão"** que a IA aplica etapa por etapa antes de responder.
- Os **textos fixos** ficam em blocos com código (ex.: `[P1-ORGANICO]`). A IA tem que copiá-los exatamente, mudando apenas o `[NOME]`, e conferir linha por linha antes de responder.
- Há uma **lista fechada de emojis** permitidos, para impedir que a IA troque um emoji por outro parecido.
- Ordem do prompt: instruções → roteiro → nome do cliente → origem do contato → histórico → mensagem atual → regras de cópia.

**Padrão de nomenclatura de `etapa`** (vários filtros dependem dele, então é preciso respeitá-lo ao editar o roteiro):
- Primeira mensagem / menu ainda sem resposta → contém `passo1` (ou `estudante_periodo` / `estudante_estagio` nas perguntas de qualificação).
- Resposta ao menu → contém `passo2`.
- Resposta ao follow-up → contém `passo3`.
- Continuação depois disso (matrícula, dúvidas) → contém `passo4`.

### 1.3 Data Store

> Todos os "assuntos" ficam numa única estrutura e são diferenciados pela **key** de cada registro.

| Key | Conteúdo | Como é criada |
|---|---|---|
| `roteiro_whatsapp` | Cache do roteiro (Google Docs) | Automática |
| `atual` | Os 6 links reais + `pdf_media_id` do PDF | **Manual**, editando direto na aba Records |
| `<telefone>` | Histórico, status e dados de follow-up do lead | Automática, na primeira mensagem |

<details>
<summary>Campos da estrutura</summary>

| Campo | Tipo | Função |
|---|---|---|
| `link01`–`link06` | Text | Links reais (vídeo, canal, PDF, inscrição, curso on-line, Instagram) |
| `pdf_media_id` | Text | ID da mídia do PDF já hospedada na Meta |
| `conteudo` / `atualizado_em` | Text / Date | Cache do roteiro |
| `mensagens` | Text | Histórico da conversa (`Cliente:` / `Eduardo:`) |
| `status` | Text | `transbordo_humano` ou vazio |
| `nome_cliente` | Text | Primeiro nome do lead |
| `perfil_followup` / `elegivel_followup` | Text | Perfil e elegibilidade usados no follow-up |
| `passo1_enviado_em` / `passo2_enviado_em` / `pdf_enviado_em` | Date | Marcos de tempo de cada etapa |
| `followup_passo1_enviado` / `followup_enviado` / `followup_pdf_enviado` | Text | Controle `SIM`/`NAO` para não repetir follow-ups |

</details>

> ⚠️ **O cache do roteiro não expira sozinho.** Se o Google Doc for editado, é preciso apagar o registro `roteiro_whatsapp` (Data Stores → Records) para a mudança valer.

### 1.4 Mapa de módulos principais

<details>
<summary>Clique para expandir a tabela completa de módulos</summary>

| ID | Módulo | Função |
|---|---|---|
| 88 | Watch Events (webhook) | Gatilho: recebe os eventos da WhatsApp Cloud API |
| 227 / 228 / 230 | Sheets Search Rows + Router + AddRecord | Lista de números não conversáveis → transbordo silencioso |
| 149 / 150 / 152 | ExistRecord + IfElse + GetRecord | Lê o status do telefone, protegido contra telefone novo |
| 157 | Router | Bloqueia **qualquer** mensagem se `status = transbordo_humano` |
| 10 / 296 | Router + AddRecord | Mensagem que não é texto → transbordo, sem passar pela IA |
| 3 | Sheets Search Rows | Verifica se o telefone é de aluno matriculado |
| 42 | Router | Separa "é aluno" de "não é aluno" |
| 20 / 25 / 18 / 17 / 19 / 63 | ExistRecord + IfElse + Get/AddRecord + Google Docs | Cache do roteiro |
| 103 / 104 / 85 / 341 | ExistRecord + IfElse + GetRecord + Set Variable | Leitura protegida do histórico |
| 344 | **Anthropic Claude – Create a Message** | Gera a resposta com JSON Schema |
| 351 | Resume (error handler do 344) | *Fallback*: resposta segura de transbordo se a API falhar |
| 350 / 352 | Parse JSON + Resume | Parse da resposta + *fallback* se o JSON vier inválido |
| 80 / 73 | GetRecord + Set Variable | Busca os links reais e troca os marcadores `Link 0X` |
| 74 / 75 / 89 | Router + AddRecord + Send Message | Transbordo (decisão da IA ou palavra-chave) |
| 76 | AddRecord | Grava histórico, perfil e horários das etapas para o follow-up |
| 179 / 90 / 180 | Router + Send Message | Resposta normal ou mensagem de encerramento |
| 168 / 127 / 287 / 278 | Router + Send Document + Sleep + Send Message | PDF de investimento seguido de "à disposição" |
| 170 | Send Message | "À disposição" para treinador / estudante não elegível |
| 113 / 114 / 95 / 97 / 44 / 101 | Leitura protegida + Router + Send + AddRecord | Reconhecimento do aluno + transbordo imediato |

</details>

### 1.5 Cenário de follow-up

Cenário agendado que percorre o Data Store em busca de leads que pararam de responder e manda **uma única** retomada por etapa.

**Filtros de elegibilidade:**
- O número **não** está na lista de não conversáveis e o lead **não** está em transbordo humano.
- Horário entre **7h e 21h**, para não mandar mensagem de madrugada.
- O follow-up daquela etapa ainda não foi enviado (`NAO`).

| Situação do lead | Espera | Mensagem |
|---|---|---|
| Parou no **Passo 1** (não respondeu ao menu) | 4h | Convite para retomar a conversa sobre o Spine Focus |
| **Veterinário** ou **estudante elegível** que já recebeu o PDF | 8h | "O que achou da proposta da imersão?" (com "Dr(a)." para veterinários) |
| **Treinador** | 4h | "O que achou das nossas redes sociais?" |
| **Estudante não elegível** | 4h | "O que achou do curso on-line de Anatomia e Biomecânica?" |

Depois do envio, o cenário acrescenta a mensagem ao histórico e marca o follow-up como `SIM`. Assim, a resposta do lead volta para o agente com o contexto completo e a IA reconhece a etapa `passo3`.

### 1.6 Regras críticas de segurança

- **Fidelidade textual**: os textos fixos do roteiro são copiados exatamente. A IA nunca parafraseia.
- **Proibição de diagnóstico**: a IA nunca faz avaliação clínica nem recomenda tratamento.
- **Mídia nunca chega à IA**: áudio e imagem vão direto para o atendimento humano.
- **Links nunca são gerados pela IA**: ela só escreve marcadores, e a URL real entra depois, de forma determinística.
- **Nunca pular etapa sem resposta**: só classifica um estudante como elegível ou não elegível quando as duas perguntas de qualificação têm resposta real e visível no histórico.
- **Transbordo com redundância**: decisão da IA + palavra-chave + *fallback* de erro. Em qualquer um dos casos, o bot fica em silêncio dali em diante.

### 1.7 Checklist de testes de ponta a ponta

1. Cliente novo vindo de anúncio → recebe a abertura de anúncio do roteiro.
2. Cliente novo orgânico → recebe a abertura orgânica e o menu.
3. Continuidade do histórico (a IA lembra do contexto anterior).
4. Pedido explícito para falar com um humano → transbordo, e o bot silencia depois.
5. Estudante: as duas perguntas de qualificação são feitas antes da classificação.
6. Veterinário pede investimento → recebe o PDF seguido de "à disposição".
7. Aluno matriculado → reconhecimento + transbordo, e silêncio a partir da segunda mensagem.
8. Mensagem que não é texto → transbordo sem passar pela IA.
9. Número da lista de não conversáveis → nenhuma resposta.
10. Follow-up: cada perfil recebe a mensagem certa no tempo certo, só uma vez e dentro do horário permitido.

---

## 📲 Projeto 2: Automação de Conteúdo (Instagram/YouTube → Grupos WhatsApp)

Sistema que capta as publicações do Instagram e os vídeos do YouTube da Axial e os distribui de forma controlada e segmentada nos grupos de WhatsApp da comunidade, com regras de cadência para evitar banimento.

### 2.1 Visão geral

| | |
|---|---|
| **Fontes** | Instagram (@axialacademy, pela API oficial) + canal do YouTube (RSS) |
| **Destino** | Grupo da comunidade Axial Academy + grupo de testes (homologação) |
| **Tipos de mídia** | Vídeo/Reels, imagem e carrossel |
| **Cadência de envio** | 1 disparo a cada 4 dias (96h), sempre o item mais antigo da fila |
| **Segmentação** | Por grupo, definida por conteúdo na planilha |

### 2.2 Stack tecnológica

| Ferramenta | Função |
|---|---|
| **Make** | Motor de automação: orquestra a captação, a fila e o disparo |
| **Instagram for Business (Graph API)** | Gatilho oficial de novos posts (Watch Media) |
| **YouTube RSS nativo** | Feed público do canal, sem ferramenta extra |
| **Cloudinary** | Hospeda de forma permanente as mídias do Instagram (as URLs originais expiram) |
| **Google Sheets** | Fila de controle de todo o conteúdo |
| **Make Data Store** | Liga os posts do Instagram que anunciam um vídeo do YouTube |
| **Z-API** | Envio para os grupos de WhatsApp (número dedicado). Imagens pelo módulo nativo e vídeos pela API HTTP |

### 2.3 Arquitetura

```text
┌───────────────────────┐        ┌───────────────────────┐
│  Instagram (Graph API) │        │     YouTube (RSS)      │
└───────────┬───────────┘        └───────────┬───────────┘
            ▼                                ▼
   Captação - Instagram              Captação - YouTube
            │                                │
     Post menciona YouTube?                  │
      ├─ SIM → Data Store (legenda           │
      │        guardada pela data)           │
      └─ NÃO → Monta a mensagem              │
               │                             │
          Tipo de mídia                      │
       ┌───────┴───────┐                     │
     VIDEO      IMAGE/CAROUSEL               │
       └──► Cloudinary ◄┘                    │
               │                             │
               ▼                             ▼
        ┌─────────────────────────────────────────┐
        │     Google Sheets (fila de conteúdo)     │
        └────────────────────┬────────────────────┘
                             ▼
                 Cenário de Disparo (a cada 4 dias)
                 Item "Pendente" mais antigo
                             │
                          Router
              ┌──────────────┴──────────────┐
        Grupo Axial (D = SIM)       Grupo Teste (E = TESTE)
              │                             │
     YouTube? → busca a legenda             │
     do Insta no Data Store e               │
     enriquece a mensagem                   │
              │                             │
     VIDEO → Z-API send-video      VIDEO → Z-API send-video
     IMAGEM → Z-API send-image     IMAGEM → Z-API send-image
              │                             │
        Sleep 15s → Update Row (Status = "Enviado", Data_envio)
```

### 2.4 Estrutura da planilha (Google Sheets)

| Coluna | Campo | Descrição |
|---|---|---|
| A | Origem | `Instagram` ou `Youtube` |
| B | Link | URL do post/vídeo |
| C | Data_captação | `YYYY-MM-DD` (define a ordem da fila) |
| D | Grupo 1 | `SIM` envia para o grupo da comunidade |
| E | Grupo 2 | `TESTE` envia para o grupo de homologação |
| F | Status | `Pendente` ou `Enviado` |
| G | Data_envio | Preenchida automaticamente após o disparo |
| H | Título | Título do vídeo (YouTube) |
| I | Mensagem pronta | Texto final formatado para o WhatsApp |
| J | Link Imagem/Vídeo | URL da mídia (Cloudinary ou thumbnail do YouTube) |
| K | Tipo | `VIDEO`, `IMAGE` ou `CAROUSEL_ALBUM` |
| L | Publicado_em | Data e hora da publicação original |

### 2.5 Regras de negócio

1. **Fila única**: o conteúdo captado automaticamente e o inserido à mão entram na mesma fila.
2. **Cadência controlada**: 1 disparo a cada 4 dias, sempre o item pendente mais antigo.
3. **Segmentação por grupo**: cada conteúdo só vai para os grupos marcados na planilha.
4. **Sem duplicidade entre fontes**: um post do Instagram que anuncia um vídeo do YouTube não é disparado sozinho. A legenda é guardada e incorporada à mensagem do vídeo.
5. **Mídia permanente**: as mídias do Instagram são copiadas para o Cloudinary antes de entrar na fila, porque os links da Meta expiram.
6. **Envio por tipo**: vídeos saem como vídeo nativo e imagens/carrosséis como imagem com legenda.
7. **Delay de segurança**: 15 segundos entre envios.
8. **Ambiente de testes**: um grupo separado permite validar mensagens novas antes de chegar à comunidade.
9. **Número dedicado**: a Z-API roda num número de WhatsApp separado dos números pessoal e comercial.

### 2.6 Cenários no Make

| Cenário | Gatilho | Função |
|---|---|---|
| Captação - Instagram | Instagram for Business – Watch Media | Capta o post, hospeda a mídia e grava na fila |
| Captação - YouTube | RSS – Watch Feed Items | Capta o vídeo e grava na fila |
| Disparo | Agendado (a cada 4 dias) | Lê a fila, monta e envia as mensagens, atualiza o status |

**Adicionando conteúdo antigo à mão:** abra a planilha → crie uma linha → preencha Origem, Link, Mensagem pronta e Link da mídia → marque os grupos → Status = `Pendente`. O item entra na fila normalmente, ordenado pela Data_captação.

### 2.7 Troubleshooting comum

| Problema | Causa provável |
|---|---|
| Mesmo item captado várias vezes | Erro num módulo posterior impediu o checkpoint do gatilho de avançar |
| Imagem/vídeo não carrega no WhatsApp | URL original do Instagram expirou (confira se o upload no Cloudinary rodou) |
| Mensagem não chega ao grupo | ID do grupo incorreto ou instância Z-API desconectada |
| Fila em ordem errada | Data_captação fora do formato `YYYY-MM-DD` |
| Erro 415 na Z-API | Falta o header `Content-Type: application/json` |
| Vídeo do YouTube sem a legenda do Instagram | O post do Instagram foi publicado em outra data ou não menciona "YouTube" |

---

## 🔗 Recursos compartilhados

Os dois projetos rodam na **mesma conta do Make**:

- **Créditos somados**: o Eduardo (agente + follow-up) e a automação de conteúdo (3 cenários) gastam do **mesmo pool mensal** de créditos. O plano tem que ser dimensionado pelo volume dos dois juntos.
- **Cenários ativos**: são 5 no total. Isso passa do limite do plano gratuito, por isso a conta usa o plano **Core** (pago).
- **Data Stores**: cada projeto tem o seu (o "Data Store" do Eduardo e o "Data store grupos" da automação de conteúdo), e os escopos não se misturam.
- **Google**: o Projeto 1 usa Google Docs e duas planilhas (inscrições confirmadas e números não conversáveis). O Projeto 2 usa a planilha da fila de conteúdo.

## 💰 Custos mensais

| Item | Custo estimado |
|---|---|
| Make (plano Core, compartilhado) | R$ 50 |
| Z-API (Projeto 2) | R$ 99 |
| Anthropic Claude API (Projeto 1) | Cobrança por uso (tokens), proporcional ao volume de conversas |
| Cloudinary (Projeto 2) | Gratuito no volume atual (free tier) |
| WhatsApp Cloud API (Projeto 1) | Mensagens de atendimento gratuitas na janela de 24h. Fora dela, cobra conforme a tabela da Meta |
| **Total fixo estimado** | **~R$ 150/mês + uso da API de IA** |

## 🚀 Próximos passos

Os dois projetos estão **temporariamente concluídos** e funcionando em produção, mas há **novas atualizações e expansões a caminho**:

- [ ] **Follow-up pós-PDF**: retomada específica para quem recebeu o PDF de investimento e não respondeu (a estrutura de dados já está pronta).
- [ ] **Mais fontes e grupos**: mais contas de Instagram e mais grupos segmentados por público na automação de conteúdo.
- [ ] **Botões interativos**: templates com botões no lugar dos menus numerados e dos links em texto.
- [ ] **Expiração automática do cache** do roteiro.
- [ ] **Log e auditoria** das conversas e dos disparos numa tabela separada, para métricas de conversão.

## 🔑 Acessos e propriedade

Todas as contas (Make, Z-API, Anthropic, Cloudinary, Google, Meta Business) devem estar no nome e no e-mail da **Axial Academy**, não da desenvolvedora. Isso garante a continuidade caso o responsável técnico mude no futuro.

> 🔐 As credenciais, tokens, IDs de grupos e IDs de planilhas foram omitidos desta documentação de propósito.

---

<div align="center">

*Desenvolvido por Julia Bezerra · Última atualização: novembro/2026*

</div>
