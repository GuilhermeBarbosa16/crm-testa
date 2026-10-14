# crm-testa — Motor de Agentes de IA + Automação (planejamento)

Projeto de estudo e **planejamento de arquitetura** para construir, em Node.js/TypeScript,
um motor de **agentes de IA conversacionais + automação** para CRM, com capacidades
equivalentes às do recurso **"Agente de IA"** do Kommo.

> **Status:** só planejamento. Nenhuma função de produção implementada ainda.
> Este repositório para na fase de plano de execução (conforme combinado).

## Escopo e postura (clean-room)

- O estudo observou o recurso de Agente de IA do Kommo numa **conta autorizada** (login próprio no Chrome)
  e a **documentação pública** do Kommo, para catalogar **o que o recurso faz** (capacidades e conceitos).
- O desenho é **clean-room**: replicamos *capacidades e conceitos* (ideias de produto, que não são protegíveis),
  **não** o código proprietário do Kommo. Nada foi descompilado nem copiado da implementação deles.
- Onde fizer sentido **integrar** com o Kommo em vez de clonar, usamos a **API pública oficial** do Kommo
  (OAuth 2.0, webhooks, chats, Salesbot). Ver `docs/02-arquitetura.md`.

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [`docs/01-estudo-kommo.md`](docs/01-estudo-kommo.md) | Inventário do recurso Agente de IA do Kommo: conceitos, UI, API pública |
| [`docs/02-arquitetura.md`](docs/02-arquitetura.md) | Arquitetura Node proposta: componentes, modelo de dados, stack, trade-offs |
| [`docs/03-plano-execucao.md`](docs/03-plano-execucao.md) | Plano de execução em fases (o que construir, em que ordem, critérios de pronto) |

## Próximo passo

Revisar o plano em `docs/03-plano-execucao.md` e decidir **caminho**:
**(A)** motor independente (clean-room) ou **(B)** camada de automação que integra na API do Kommo —
ou **(C)** híbrido (motor próprio + adaptador Kommo). Só então começa a implementação.
