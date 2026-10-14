# 03 — Plano de execução (fases)

Ordem de construção do motor descrito em `02-arquitetura.md`.
Cada fase tem **objetivo**, **entregas** e **critério de pronto (DoD)**.
Este documento é o **ponto de parada**: planejamento pronto, implementação começa só após decisão do caminho (A/B/C).

Princípio: **vertical slices** — cada fase entrega algo testável ponta-a-ponta, não camadas soltas.

---

## Fase 0 — Scaffold
- **Objetivo:** esqueleto do projeto rodando.
- **Entregas:** TS + Fastify + Drizzle + Postgres(pgvector) + Redis(BullMQ) via docker-compose; `.env.example`;
  healthcheck; lint/format; estrutura de pastas (`src/{adapters,gateway,orchestrator,actions,rag,flows,crm,db}`).
- **DoD:** `docker compose up` sobe DB+Redis; servidor responde `/health`; migration inicial aplica.

## Fase 1 — Modelo de dados + CRUD de agente/persona
- **Objetivo:** criar e configurar um agente.
- **Entregas:** tabelas da seção 4; endpoints CRUD de `agents` e `personas`; seed de um agente de exemplo.
- **DoD:** criar agente + persona via API e ler de volta; validação de schema.

## Fase 2 — Canal Webchat + Gateway (eco)
- **Objetivo:** mensagem entra e sai ponta-a-ponta.
- **Entregas:** `ChannelAdapter` webchat (HTTP + WebSocket); Gateway normaliza, cria `conversation`+`messages`,
  enfileira no BullMQ; worker responde eco; **debounce** (pausa antes da resposta) implementado.
- **DoD:** enviar msg no webchat → recebe resposta; mensagens persistidas; debounce agrupa msgs rápidas.

## Fase 3 — Resposta por LLM + Persona Compiler
- **Objetivo:** resposta real do agente (sem base ainda).
- **Entregas:** Persona Compiler (system prompt a partir dos campos); chamada ao LLM via AI SDK
  (provider por env); tom/tamanho/idioma aplicados; preview sandbox usando a mesma pipeline.
- **DoD:** agente responde no tom/idioma configurados; preview funciona sem efeitos externos.
- **Bloqueio conhecido:** precisa de chave de LLM válida (ver pendência em `02-arquitetura.md §5`).

## Fase 4 — Fontes + RAG (grounding)
- **Objetivo:** agente responde a partir da base de conhecimento.
- **Entregas:** ingestão (upload/texto → chunk → embeddings → pgvector); recuperação top-k filtrada por fontes
  ativas; prompt com grounding estrito; toggle ligar/desligar fonte.
- **DoD:** pergunta coberta pela fonte → resposta correta citando a base; fora da base → "não sei"/transfere.

## Fase 5 — Motor de Ações (WHEN/DO/MORE)
- **Objetivo:** comportamento guiado por regras.
- **Entregas:** representação de ações em JSONB + executor; avaliadores WHEN (`intent`, `sentiment`,
  `inactivity` via job atrasado); executores DO (`reply`, `request_info`/slot filling, `update_lead`,
  `transfer_to_human`); guardas MORE; estado da conversa (slots/transfer).
- **DoD:** cenário "cliente pede orçamento → agente solicita dados → atualiza lead → responde" roda fim-a-fim;
  timer de inatividade dispara ação e é cancelado se o cliente responde.

## Fase 6 — Fluxos / Salesbot runner + transfer p/ humano
- **Objetivo:** automações scriptadas e handoff.
- **Entregas:** `flows` (definição JSONB) + runner com estado (`flow_runs`); `start_bot` chama o runner;
  `transfer_to_human` silencia agente, marca `assigned_to`, notifica.
- **DoD:** uma ação inicia um fluxo de 3 passos; transferência tira o agente da conversa corretamente.

## Fase 7 — Voz (STT)
- **Objetivo:** aceitar notas de voz.
- **Entregas:** detecção de áudio no inbound → download → transcrição (provider plugável) → entra na pipeline normal.
- **DoD:** áudio no webchat vira texto e é respondido com contexto.

## Fase 8 — Canais reais + (opcional) adaptador Kommo
- **Objetivo:** sair do webchat.
- **Entregas:** adaptador **Telegram** e/ou **WhatsApp Cloud API** (inbound webhook + outbound);
  **opcional** adaptador **Kommo** (OAuth 2.0, webhook de mensagem, enviar msg, `launchBot`→Salesbot,
  `updateLead`) implementando `CrmPort`/`ChannelAdapter`.
- **DoD:** conversa real por um canal externo responde pelo motor; (se Kommo) webhook→resposta→update no Kommo.
- **Bloqueio conhecido:** credenciais de canal + App OAuth Kommo (ver pendências).

## Fase 9 — Metering, guardrails, observabilidade
- **Objetivo:** pronto para uso controlado.
- **Entregas:** `usage_events` (tokens/custo por agente); limites + alertas; rate limit por conversa;
  teto de passos/turn; logs estruturados + traços; painel básico (métricas da aba "Painel").
- **DoD:** uso é contabilizado e limitável; loops e abusos barrados; dashboards mínimos.

---

## Matriz de cobertura (capacidade → fase)

| Capacidade (do estudo §5) | Fase |
|---|---|
| Multi-agente + atribuição a canais | 1, 2, 8 |
| Persona (tom/tamanho/idioma/pausa/diretrizes) | 1, 3 |
| Fontes + grounding (RAG) | 4 |
| Ações WHEN/DO/MORE | 5 |
| DO: solicitar info / atualizar lead / iniciar bot / transferir | 5, 6 |
| Canais plugáveis | 2, 8 |
| Conversas com estado | 2, 5 |
| Voz | 7 |
| Preview/sandbox | 3 |
| Metering + guardrails | 9 |
| Integração Kommo (opcional) | 8 |

## Pendências que exigem decisão/insumo do Guilherme (antes de implementar)
1. **Caminho A / B / C** (motor próprio, camada sobre Kommo, ou híbrido). Define Fase 8.
2. **Chave de LLM válida** (as locais estão revogadas) — ou usar AI Gateway. Bloqueia Fase 3+.
3. **Canais-alvo** do MVP (webchat basta pra validar; WhatsApp/Telegram/Kommo depois).
4. **ORM:** Drizzle (recomendado) vs Prisma.

## Estimativa grosseira
MVP conversacional ancorado (Fases 0–5) é o núcleo de valor. Fases 6–9 incrementam.
Sem implementar nada ainda — este é o escopo a aprovar.
