## 2026-10-14 — Postura clean-room no estudo do Kommo
- **Decisão:** replicar capacidades/conceitos do Agente de IA do Kommo, não o código deles; integração via API pública oficial quando for o caso.
- **Por quê:** ideia de produto não é protegível, implementação é; evita risco jurídico e é melhor engenharia.
- **Descartado:** descompilar o frontend do Kommo (skill REA) para copiar implementação — desnecessário para planejar e arriscado.

## 2026-10-14 — Arquitetura híbrida (caminho C) como alvo
- **Decisão:** motor de agentes no centro, Kommo (e demais canais/CRM) como adaptadores plugáveis atrás de interfaces (ChannelAdapter, CrmPort).
- **Por quê:** o mesmo código serve motor independente (A), camada sobre Kommo (B) ou híbrido; troca por configuração.
- **Descartado:** acoplar direto na API do Kommo (B puro) — travaria o produto no Kommo.

## 2026-10-14 — Ações como dados (JSONB interpretado)
- **Decisão:** regras WHEN/DO/MORE armazenadas em JSONB e executadas por um interpretador, não hardcoded.
- **Por quê:** permite editar ações por UI no futuro (como o Kommo) sem deploy; espelha o modelo do produto estudado.
- **Descartado:** cada ação como código fixo — inflexível, exige deploy a cada regra.

## 2026-10-14 — Stack: Fastify + Postgres(pgvector) + Redis/BullMQ + AI SDK
- **Decisão:** Node22/TS, Fastify, Postgres com pgvector (RAG no mesmo banco), BullMQ para async/timers, AI SDK provider-agnóstico (Claude padrão), Drizzle como ORM.
- **Por quê:** BullMQ dá timers de inatividade de graça; pgvector evita infra de vetores separada no início; AI SDK troca provedor por env.
- **Descartado:** Qdrant/pinecone já de início (overkill p/ MVP); LangChain (abstração a mais p/ o que precisamos).
