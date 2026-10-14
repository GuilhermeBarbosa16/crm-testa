# 02 — Arquitetura proposta (Node.js / TypeScript)

Desenho clean-room para replicar as capacidades do estudo (`01-estudo-kommo.md`) em Node.
Foco em um **motor de agentes + automação** desacoplado dos canais e do provedor de LLM.

---

## 1. Três caminhos possíveis (decisão a tomar antes de implementar)

| Caminho | O que é | Quando faz sentido |
|---|---|---|
| **A — Motor independente** | CRM/agente próprio, canais próprios | Produto autônomo, sem depender do Kommo |
| **B — Camada sobre o Kommo** | Automação Node que fala com a API do Kommo (OAuth/webhooks/Salesbot) | Já usa Kommo e só quer estender/orquestrar |
| **C — Híbrido** | Motor próprio + **adaptador** Kommo como mais um canal/CRM | Flexível; migra do Kommo aos poucos |

Recomendação: **arquitetar como C** (motor no centro, Kommo como adaptador plugável).
Assim o caminho A e B saem do mesmo código, trocando adaptadores.

## 2. Visão em camadas

```
┌─────────────────────────────────────────────────────────────┐
│  Canais (adapters)   WhatsApp · Telegram · Webchat · Email    │
│                      · Kommo(API) · Preview/Sandbox           │
└───────────────┬───────────────────────────────┬──────────────┘
         inbound │                               │ outbound
┌───────────────▼───────────────────────────────▼──────────────┐
│                    Gateway de Mensagens                        │
│   normaliza evento → fila (BullMQ) → despacha ao Orchestrator  │
└───────────────┬───────────────────────────────────────────────┘
                │
┌───────────────▼───────────────────────────────────────────────┐
│                     Agent Orchestrator                         │
│  carrega config do agente p/ canal → roda motor de Ações →     │
│  monta prompt (Persona + RAG das Fontes) → gera resposta LLM   │
└──┬──────────┬───────────────┬──────────────┬──────────────┬────┘
   │          │               │              │              │
┌──▼───┐ ┌────▼─────┐ ┌───────▼──────┐ ┌─────▼─────┐ ┌──────▼─────┐
│Persona│ │ Motor de │ │  RAG / Fontes │ │ Fluxos /  │ │ CRM / Leads│
│Compiler│ │  Ações   │ │ (pgvector)    │ │ Salesbot  │ │ (store/API)│
└───────┘ │WHEN/DO/MORE│ └──────────────┘ │  runner   │ └────────────┘
          └──────────┘                    └───────────┘
┌───────────────────────────────────────────────────────────────┐
│  Infra: PostgreSQL(+pgvector) · Redis(BullMQ) · provider LLM    │
│  · STT(voz) · Observabilidade/metering                          │
└───────────────────────────────────────────────────────────────┘
```

## 3. Componentes

### 3.1 Channel Adapters (plugáveis)
Interface única normalizada; cada canal implementa `receive()` (webhook→evento interno) e `send()`.
```ts
interface ChannelAdapter {
  id: string;                       // 'whatsapp' | 'telegram' | 'webchat' | 'kommo' ...
  verifyWebhook(req): boolean;
  parseInbound(req): InboundMessage; // → {channel, externalChatId, contact, text|audio, ts}
  sendOutbound(msg: OutboundMessage): Promise<void>;
  setTyping?(chatId, on): Promise<void>;
}
```
Primeiro canal a implementar: **Webchat** (mais simples, serve de sandbox/preview também).

### 3.2 Gateway de Mensagens
- Recebe inbound normalizado, resolve/cria **Conversation**, aplica **debounce** (a "pausa antes da resposta":
  agrupa mensagens rápidas do cliente), enfileira job no **BullMQ**.
- Saída: idempotência por `externalMessageId`, retry/backoff.

### 3.3 Agent Orchestrator (coração)
Para cada mensagem/turn:
1. Resolve o **agente** ativo para aquele canal.
2. Carrega estado da conversa (slots pendentes, transferências, bot em execução).
3. Roda o **Motor de Ações** (ver 3.4) para decidir o comportamento.
4. Se precisa responder: **Persona Compiler** + **RAG** montam o prompt → chama o LLM → grava e envia.
5. Registra **uso/metering**.

### 3.4 Motor de Ações (WHEN / DO / MORE)
- **WHEN (avaliadores):**
  - `message_intent` — classificação de intenção/palavra-chave (LLM classifier ou regra).
  - `sentiment` — análise de sentimento (LLM).
  - `inactivity` — timer: job BullMQ **atrasado** agendado ao fim do turn; se o cliente responde, cancela.
- **DO (executores sequenciais):**
  - `reply` — resposta ancorada (sempre presente).
  - `request_info` — slot filling: define slot esperado, pausa a ação até o cliente informar.
  - `update_lead` — escreve no CRM store (ou API Kommo).
  - `start_bot` — chama o **Fluxo/Salesbot runner**.
  - `transfer_to_human` — marca conversa p/ atendimento humano, notifica, silencia o agente.
- **MORE:** condições/guardas por ação (ex.: só continuar quando `slot.X` preenchido).
- Representação: ações como **dados** (JSONB) interpretados por um executor — não código hardcoded.
  Permite editar ações pela UI no futuro, como no Kommo.

### 3.5 Persona Compiler
Monta o **system prompt** a partir dos campos da Persona (função, tom, tamanho, idioma, diretrizes)
+ regras de guardrail ("responda apenas com base nas Fontes; se não houver, diga que não sabe / transfira").

### 3.6 RAG / Fontes
- Ingestão: upload/URL → extração → **chunking** → **embeddings** → `source_chunks` (pgvector).
- Consulta: no turn, recupera top-k por similaridade (filtrado por fontes **ativas** do agente) → injeta no prompt.
- Grounding estrito: prompt instrui a responder só a partir do contexto recuperado (como o Kommo).

### 3.7 Fluxos / Salesbot runner
- Fluxos scriptados (sequência de passos: enviar msg, aguardar, condição, setar campo, chamar webhook).
- Acionados por `start_bot` (DO) ou por gatilho próprio. Estado persistido por conversa.

### 3.8 CRM / Leads
- Store próprio (Postgres) **ou** delegado via **adaptador** (ex.: Kommo API) atrás de uma interface `CrmPort`.
```ts
interface CrmPort {
  upsertContact(c): Promise<ContactId>;
  updateLead(id, patch): Promise<void>;
  launchBot?(botId, entity): Promise<void>; // mapeia p/ Salesbot do Kommo no adaptador
}
```

### 3.9 Voz (STT)
- No inbound de áudio: baixa mídia → **transcrição** (provider STT / Whisper) → vira texto e segue o fluxo normal.

### 3.10 Preview / Sandbox
- Canal especial `preview`: mesma pipeline, sem efeitos externos (não grava no CRM real, não envia a canais).
  Serve ao "testar o que eu sei" visto na UI do Kommo.

### 3.11 Metering & Guardrails
- Contabiliza tokens/ações por agente/conta (equivale aos "créditos"). Limites e alertas.
- Guardrails: rate limit por conversa, bloqueio de loop, teto de passos por turn, PII básica.

## 4. Modelo de dados (Postgres)

```
agents(id, account_id, name, status, created_at)
personas(agent_id PK/FK, role_prompt, tone, reply_length, language, reply_delay_s, guidelines jsonb)
sources(id, agent_id, type, title, enabled, meta jsonb, created_at)
source_chunks(id, source_id, content, embedding vector, tokens)         -- pgvector
actions(id, agent_id, order, when jsonb, do jsonb, more jsonb, enabled)
channels(id, account_id, kind, credentials_ref, config jsonb)
agent_channels(agent_id, channel_id)                                     -- N:N
conversations(id, agent_id, channel_id, external_chat_id, contact_id,
              state jsonb, assigned_to, status, updated_at)              -- state: slots/transfer/bot
messages(id, conversation_id, direction, kind, text, media_ref, ext_id, ts)
contacts(id, account_id, name, phones jsonb, emails jsonb, fields jsonb)
leads(id, account_id, contact_id, stage, fields jsonb)
flows(id, agent_id, name, definition jsonb)                              -- Salesbot runner
flow_runs(id, flow_id, conversation_id, cursor, state jsonb, status)
usage_events(id, account_id, agent_id, conversation_id, kind, tokens, cost, ts)
```

## 5. Stack escolhida (e por quê)

| Camada | Escolha | Razão / trade-off |
|---|---|---|
| Runtime | **Node 22+ / TypeScript** | Já é o ambiente; tipos ajudam no executor de ações |
| HTTP/Webhooks | **Fastify** | Rápido, schema validation nativo (JSON Schema/Zod) |
| Fila/async/timers | **BullMQ (Redis)** | Jobs atrasados = timers de inatividade; retry/backoff p/ canais |
| DB | **PostgreSQL** | Relacional + JSONB p/ ações/flows como dados |
| Vetores | **pgvector** | Fica no mesmo Postgres (simplicidade); trocável por Qdrant se escalar |
| ORM | **Drizzle** (ou Prisma) | Drizzle = SQL-first, migrations simples; Prisma se preferir DX |
| LLM | **AI SDK (Vercel) provider-agnóstico** | Claude (padrão) / OpenAI por env; troca sem reescrever |
| STT | provider plugável (Whisper/API) | Voz é fase tardia; interface isolada |
| Realtime | **WebSocket** (ws) | Preview ao vivo + push de eventos de canal |

### Dependências externas / segredos (pendências)
- Chave de LLM (ex.: `ANTHROPIC_API_KEY`). **Nota:** as chaves locais atuais estão revogadas (401) —
  precisará de chave nova ou AI Gateway antes da fase de LLM.
- Credenciais por canal (WhatsApp Cloud API, Telegram bot token, etc.).
- Para o caminho B/C: **App OAuth no Kommo** (client id/secret, redirect) + registro de webhook.

## 6. O que fica fora (não-objetivos nesta fase)
- UI de administração (construir só a API/motor primeiro; UI depois).
- Billing real de créditos (só metering interno).
- Multi-tenant avançado / RBAC completo (modelar `account_id` desde já, mas não implementar papéis finos).
