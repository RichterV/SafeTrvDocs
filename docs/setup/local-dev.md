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
- Node.js 20+ (para o frontend Ionic, quando for criado)

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

`WS ws://localhost:8000/api/v1/trips/{trip_id}/ws?token=<access_token>` — o token vem por query param (não por header). Protocolo completo documentado em [WebSocket — rastreamento ao vivo](../system/websocket.md); já testado de ponta a ponta com clientes reais.

## Testes e lint

```bash
cd backend
uv run pytest
uv run ruff check .
```

## Documentação (este site)

```bash
uv tool run --with mkdocs-material mkdocs serve -a localhost:8001   # a partir da raiz do repositório
```

O tema Material (`mkdocs.yml`) não vem embutido no `mkdocs` puro — por isso o `--with mkdocs-material`. Porta `8001` para não colidir com a API (`8000`). `./menu.sh` tem uma opção que já roda isso.
