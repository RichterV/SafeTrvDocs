# Ambiente de desenvolvimento local

## Dois repositórios

O código (backend, `menu.sh`, `docker-compose.yml`) vive no repositório **SafeTrv**, hospedado num Gitea privado da equipe (sem espelho público). Esta documentação vive separadamente em [SafeTrvDocs](https://github.com/RichterV/SafeTrvDocs), publicada em [richterv.github.io/SafeTrvDocs](https://richterv.github.io/SafeTrvDocs/). Para desenvolver localmente, clone os dois lado a lado:

```
Projetos/
  SafeTrv/        # código — backend, menu.sh, docker-compose.yml
  SafeTrvDocs/     # esta documentação
```

`./menu.sh` (dentro de `SafeTrv/`) detecta `../SafeTrvDocs` automaticamente para a opção de servir a documentação localmente.

## Pré-requisitos

- Python 3.10+ e [uv](https://docs.astral.sh/uv/)
- Docker Engine + Compose plugin (para Postgres + PostGIS via Docker Compose) — em Linux, instalar via o repositório oficial da Docker (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`); em outros sistemas, o Docker Desktop cobre o mesmo papel
- Node.js 20+ e npm (para o frontend Ionic + Angular — testado com Node 24)

Atalho: use `./menu.sh` (raiz do repositório **SafeTrv**) para um menu interativo que cobre a maior parte dos comandos abaixo (subir/parar o ambiente, rodar testes, importar dados, servir a documentação, commit rápido).

## Backend

```bash
cd backend
uv sync                      # instala dependências (runtime + dev)
cp .env.example .env         # gere um JWT_SECRET_KEY forte e configure OPEN_ROUTE_SERVICE_API_KEY
```

`OPEN_ROUTE_SERVICE_API_KEY` é obrigatória para criar viagens (chave gratuita em https://openrouteservice.org/dev/#/signup) — sem ela, `POST /api/v1/trips` responde 502 com uma mensagem explicando o que falta.

Suba o banco de dados (raiz do repositório):

```bash
docker compose up -d db
```

Rode as migrações e suba a API:

```bash
cd backend
uv run alembic upgrade head          # aplica as migrações existentes
uv run uvicorn app.main:app --reload
```

Após o `alembic upgrade head`, importe a malha de estados/municípios/rios do Brasil (necessário para o cálculo de risco de proximidade a rio funcionar de verdade — sem isso a tabela `river_geometries` fica vazia e esse fator de risco sempre dá zero):

```bash
uv run python scripts/import_geodata.py all
```

Isso baixa dados do IBGE e da ANA/SNIRH (leva alguns minutos, principalmente os rios — ~16 páginas de requisição). Rodar de novo é seguro, sobrescreve os dados existentes. Depois de qualquer `alembic downgrade base` (que apaga todas as tabelas), rode este script de novo.

A API sobe em `http://localhost:8000` — documentação interativa em `/docs`.

### Rastreamento ao vivo (WebSocket)

`WS ws://localhost:8000/api/v1/trips/{trip_id}/ws?ticket=<ticket>` — o ticket (uso único, 30 s) vem de `POST /api/v1/trips/{trip_id}/ws-ticket`, chamado com o access token no header; o access token em si nunca vai na URL. Protocolo completo em [WebSocket — rastreamento ao vivo](../system/websocket.md).

## Frontend (Ionic + Angular)

Com a API no ar:

```bash
cd frontend
npm install        # só na primeira vez
npx ng serve --port 8100
```

Abrir `http://localhost:8100`. A porta `8100` já está liberada no CORS do backend (`cors_origins`). A URL da API fica em `src/environments/environment.ts`. `./menu.sh` opção 11 faz o mesmo. Detalhes em [Frontend](../system/frontend.md).

## Testes e lint

São três camadas. `./menu.sh` opção 7 roda as duas primeiras; a opção 12 roda o e2e.

### Backend — unitários + integração

```bash
cd backend
uv run pytest                      # tudo (50 unitários + 24 de integração)
uv run pytest -m "not integration" # só unitários (sem banco, ~2 s)
uv run ruff check .
```

Os testes de integração (`tests/integration/`) rodam contra o Postgres + PostGIS do docker-compose, num banco separado **`safetrv_test`**, que é apagado e recriado a cada execução com as migrações do Alembic. O banco de desenvolvimento não é tocado. OpenRouteService, Open-Meteo e INMET são simulados (sem rede); a consulta espacial de proximidade a rio roda de verdade no PostGIS. **Sem Postgres no ar, esses testes são pulados** (não falham) — suba com `docker compose up -d`.

### Frontend — unitários

```bash
cd frontend
npx ng test --watch=false   # Vitest (26 testes)
npx ng lint
```

### Ponta a ponta (Playwright)

```bash
cd frontend
npx playwright install chromium   # só na primeira vez
npm run e2e                        # type-check dos testes + Playwright
npm run e2e:report                 # relatório HTML da última execução
```

Sobe a API (`:8000`) e o frontend (`:8100`) automaticamente, ou reaproveita os que já estiverem rodando. Precisa do Postgres no ar e de **rede**: o e2e usa as APIs externas reais (ORS — chave em `backend/.env` —, Open-Meteo, INMET, Nominatim) e o banco de **desenvolvimento**, criando contas novas com e-mails únicos a cada execução. Em caso de falha, screenshots e trace ficam em `frontend/e2e-results/`.

## Documentação (este site)

```bash
uv tool run --with mkdocs-material mkdocs serve -a localhost:8001   # a partir da raiz do repositório
```

O tema Material (`mkdocs.yml`) não vem embutido no `mkdocs` puro — por isso o `--with mkdocs-material`. Porta `8001` para não colidir com a API (`8000`). `./menu.sh` tem uma opção que já roda isso.
