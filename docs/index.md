# SafeTrv

Plataforma de planejamento de viagens rodoviárias que cruza rotas com previsão do tempo e dados geográficos (rios, e futuramente outros eventos climáticos) para calcular um **Índice de Risco** por trajeto.

Esta documentação (`/docs`, publicada com [MkDocs](https://www.mkdocs.org/) + Material) é o ponto de referência para quem for além do código: contexto de negócio, arquitetura e guias de setup.

> Durante a fase inicial do projeto, o arquivo `CLAUDE.md` na raiz do repositório concentra as decisões de produto/negócio em detalhe e é atualizado primeiro. As páginas aqui em `/docs` são o resumo "oficial" voltado para leitura humana e serão expandidas conforme cada parte é implementada.

## Onde começar

- [Visão geral de negócio](business/overview.md)
- [Regras de negócio (linguagem não técnica)](business/rules.md)
- [Índice de risco climático](business/risk-index.md)
- [Visão geral da arquitetura](system/overview.md)
- [Referência da API](system/api-reference.md)
- [WebSocket — rastreamento ao vivo](system/websocket.md)
- [Serviços internos](system/services.md)
- [Modelo de dados](system/data-model.md)
- [Ambiente de desenvolvimento local](setup/local-dev.md)
