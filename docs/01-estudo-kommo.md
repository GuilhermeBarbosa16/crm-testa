# 01 — Estudo: recurso "Agente de IA" do Kommo

Fonte: observação da conta autorizada (`new1741262133.kommo.com`, usuário logado) +
documentação pública do Kommo (`support.kommo.com`, `developers.kommo.com`, blog oficial).
Objetivo: catalogar **capacidades e conceitos** do recurso para desenhar um equivalente clean-room em Node.

---

## 1. Visão geral do produto

O "Agente de IA" do Kommo é um **assistente conversacional** que responde mensagens de clientes
dentro do CRM, ancorado numa base de conhecimento, com comportamento guiado por regras e uma persona.

Observado na conta:
- Seção **Agente de IA** no menu lateral (ao lado de Comunicações, Pipelines, Automações, Insights).
- **Multi-agente:** lista "Meus agentes" + aba "Explorar" (modelos prontos). É possível ter vários agentes,
  cada um independente, com coluna de **Canais**, **Última atividade** e **Status** (ativo/inativo).
- **Cobrança por créditos:** indicador "Créditos usados 0 / 2.250" + botão "Comprar créditos".
  Uso é medido (metering) — relevante para o nosso desenho (precisamos de contabilização de uso).
- Agente existente na conta: *"Suporte ao cliente de varejo"*.

## 2. Anatomia de um agente (abas do editor)

Observado no editor do agente. Seis abas + preview:

| Aba | Função |
|---|---|
| **Persona** | Identidade e estilo do agente |
| **Ações** | Regras de comportamento (gatilho → ação) |
| **Fontes** | Base de conhecimento (o agente só responde a partir daqui) |
| **Integrações** | Conexão com canais de mensagem |
| **Painel** | Métricas / monitoramento |
| **Configurações** | Ajustes do agente |
| **Preview** (lateral) | Sandbox de teste ("Gerar questão de teste", chat ao vivo) |

Botão **"Ativar agente de IA"** controla o status.

### 2.1 Persona (observado na UI)
Campos concretos vistos:
- **Função e personalidade** — prompt livre descrevendo papel/objetivo (ex.: "Você é um assistente de
  suporte ao cliente... tom acolhedor e informado, membro confiável da equipe").
- **Tom de voz** — select (ex.: *Amigável*; também casual/profissional).
- **Tamanho das respostas** — select (*Curtas* / médias / longas).
- **Idioma da resposta** — select (*Português*; ou detecção automática).
- **Pausa antes da resposta (segundos)** — ex.: `3`. Aguarda o cliente terminar de digitar várias mensagens
  antes de responder (debounce de mensagens).
- **Diretrizes** — regras extras (palavras a evitar, pontos a enfatizar).

### 2.2 Ações — estrutura WHEN / DO / MORE (doc pública)
Cada ação = uma regra com três partes:
- **WHEN (quando):** condição de gatilho — o que o lead diz (intenção/palavra-chave), **sentimento**,
  ou **inatividade** (não respondeu em X tempo).
- **DO (faça):** passos executados em sequência. Sempre envia uma resposta; além disso pode:
  - **solicitar informação** (pedir e aguardar um dado do cliente — slot filling);
  - **atualizar o lead** (campos/estágio no CRM);
  - **iniciar um bot** (dispara um Salesbot/fluxo);
  - **transferir o chat** (handoff para humano).
- **MORE (mais):** instruções condicionais extras (ex.: "aguardar até o cliente informar X").

### 2.3 Fontes (base de conhecimento)
- Upload de conteúdo: catálogo de produtos, políticas, FAQs, documentos, info do negócio.
- **O agente só responde usando as fontes** (grounding / recuperação). Não inventa fora delas.
- Fontes podem ser **ligadas/desligadas** sem apagar, e editadas.

### 2.4 Voz
- Processa **notas de voz** recebidas: transcreve (STT) → interpreta com Fontes/Ações/Persona → responde em texto.
- (No Kommo, recurso de plano Pro.)

### 2.5 Limitação declarada
- O agente **só responde a mensagens recebidas**; **não inicia** conversas por conta própria.
  (No nosso desenho isso vira uma *decisão de produto* — podemos permitir ou não iniciativa.)

## 3. Canais / comunicação
Observado no menu: **Inbox de chat**, **Inbox de email**, **Chats da equipe**, integração **Facebook** fixada.
O Kommo conecta WhatsApp, Instagram, Telegram, webchat, email etc. O agente é **atribuído a canais**
(coluna "Canais" na lista de agentes).

## 4. API pública do Kommo (caminho de integração)
Para o cenário de **integrar** (em vez de clonar), a API oficial oferece:

- **Auth:** OAuth 2.0 (code → access/refresh token) + token de longa duração; permissões por integração.
- **Leads / Contatos / Empresas:** CRUD, batch, vínculo entre entidades, campos personalizados, tags.
- **Chats & Mensagens:** registrar canal de chat, criar chat vinculado a contato, enviar/importar/editar
  mensagens, histórico, status de entrega, indicador de digitação, reações, **webhooks** de msg entrante/saindo.
- **Salesbot API:** listar, **lançar** (launch) e **parar** bots; confirmar execução de bloco widget.
- **Webhooks:** eventos de leads, contatos, tarefas, notas, conversas, Digital Pipeline (com retry).
- **Kommo AI API:** importar fontes / gerenciar dados de treino do agente.
- **Suporte:** arquivos (mídia de conversa), eventos/atividades, usuários/papéis, VoIP, webforms, widgets, JS SDK.

Padrão típico de integração externa:
`OAuth → webhooks (gatilho) → consultar lead/contato → criar/atualizar → enviar mensagem → acionar Salesbot`.

---

## 5. Mapa de capacidades (para replicar)
Checklist do que o nosso motor precisa cobrir:

- [ ] Multi-agente, cada um com config isolada e atribuído a canais
- [ ] Persona (prompt de função, tom, tamanho, idioma, debounce/pausa, diretrizes)
- [ ] Fontes com grounding estrito (responder só a partir da base) — RAG
- [ ] Ações WHEN/DO/MORE (gatilho por conteúdo, sentimento, inatividade)
- [ ] Passos DO: responder, solicitar info (slot filling), atualizar lead, iniciar bot/fluxo, transferir p/ humano
- [ ] Canais plugáveis (webchat, WhatsApp, Telegram, email…) com entrada/saída normalizadas
- [ ] Conversas com estado (slots pendentes, transferências, execuções de bot)
- [ ] Voz: transcrição → processamento → resposta
- [ ] Preview/sandbox de teste do agente
- [ ] Metering de uso (créditos) + guardrails
- [ ] (Opcional) Adaptador de integração com a API do Kommo
