# Publicação no servidor

O SafeTrv roda no servidor pessoal (Ubuntu Server, usuário `vrserver`), no mesmo esquema do VR Finance, acessível só pela rede Tailscale em **https://vrserver.tailbdad79.ts.net:8443**. Publicado pela primeira vez em 2026-10-11.

```text
celular/PC ──HTTPS──▶ tailscale serve :8443 ──▶ nginx 127.0.0.1:8081 ─┬─ /               → /var/www/safetrv (site)
                                                                      └─ /api/, /health  → uvicorn 127.0.0.1:8001
uvicorn ──▶ Postgres + PostGIS (Docker) em 127.0.0.1:5433
```

- **Backend:** serviço systemd `safetrv-backend` (reinicia sozinho, sobe no boot), venv em `~/safetrv/backend`. O acompanhamento das viagens em andamento (`trip_monitor`) roda ligado — com o clima real.
- **Banco:** container `postgis/postgis:16-3.4`, volume `safetrv_pgdata`, compose em `~/safetrv/db/`. Só escuta em `127.0.0.1`.
- **Segredos:** `~/safetrv/backend/.env` (permissão 600), gerado no setup com senha do banco e chave JWT aleatórias; a chave do OpenRouteService é copiada do PC no primeiro deploy, pela entrada padrão do SSH (não aparece em linha de comando nem em log).
- **Portas de outros serviços do servidor**, que não são mexidas: 443/8080/8000 (VR Finance), 3002 (Gitea).

## Setup (uma vez, no servidor)

```bash
scp deploy/setup_server.sh vrserver@vrserver.tailbdad79.ts.net:~/safetrv-setup/
ssh vrserver@vrserver.tailbdad79.ts.net
sudo bash ~/safetrv-setup/setup_server.sh
```

Idempotente. Instala nginx/venv, sobe o banco, cria o `.env`, o serviço, o site do nginx, um sudoers que permite **só** iniciar/parar/reiniciar/ver o status do `safetrv-backend` sem senha, e o `tailscale serve --https=8443`.

## Deploy (do PC)

```bash
./deploy/deploy.sh completo   # site (ng build) + backend (dependências, migrações, chave do ORS, restart)
./deploy/deploy.sh frontend   # só o site
./deploy/deploy.sh backend    # só o backend
./deploy/deploy.sh dados      # dados geográficos (IBGE/ANA) + seed de demonstração — na primeira vez
./deploy/deploy.sh seed       # recria só o cenário de demonstração
./deploy/deploy.sh status     # serviço, saúde e log
```

O envio usa `rsync` por SSH (chave, sem senha); as dependências vêm de `uv export --no-dev`. Ao fim, o script confere `https://…:8443/health` e o site. O servidor roda Python 3.14 (Ubuntu 26.04).

A carga de dados é pesada para o servidor por alguns minutos (o seed calcula a distância a rio de cada trecho de 35 viagens no PostGIS); depois disso a API fica ociosa, com uma reavaliação de viagens em andamento a cada 5 minutos.

## App Android (APK)

```bash
./deploy/build-apk.sh
```

Gera `SafeTrv-AAAAMMDD-HHMMSS.apk` na raiz do repositório (ignorado pelo git), um APK de **debug** apontando para o servidor (`frontend/src/environments/environment.mobile.ts`). Para instalar: mandar o arquivo para o celular (ex: WhatsApp Web), abrir e permitir instalar de fonte desconhecida. O celular precisa estar na rede Tailscale.

Usa o JDK 21 portátil em `~/jdk` (o Java do sistema não tem compilador) e o Android SDK em `~/Android/sdk` — os mesmos do VR Finance. Detalhes do app em [Frontend → App Android](../system/frontend.md#app-android-capacitor).
