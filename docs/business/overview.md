# Visão geral de negócio

Resumo executivo — para o detalhamento completo (personas, contexto de mercado, backlog priorizado), ver `CLAUDE.md` na raiz do repositório. Para como o sistema funciona no dia a dia, sem jargão técnico, ver [Regras de negócio](rules.md).

- **Produto:** planejamento de viagens rodoviárias + navegação, com foco em antecipar risco climático (chuva, proximidade de rios com risco de alagamento; futuramente neblina, deslizamento, vento, calor extremo).
- **Escopo geográfico:** Brasil.
- **Modelo:** multi-tenant (SaaS). Cada empresa cliente é uma Conta isolada.
- **Hierarquia:** Administrador da conta → Gerente de viagem → Viajante.
- **Clientes-alvo:** empresas com frota terrestre que agendam viagens com antecedência (transportadoras, cadeia do frio, agronegócio, frotas corporativas, fretamento de passageiros, mineração/construção).
