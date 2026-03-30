# Impulso Hub (fork do Chatwoot) — Setup de Desenvolvimento

É recomendado usar **WSL/Ubuntu**, pois o Chatwoot é bem "Linux-first" (Ruby on Rails + Node + Postgres/Redis) e o WSL (especialmente o WSL 2) oferece um ambiente Linux real e compatível, sem precisar de uma VM completa.

## Compatibilidade com a stack

O Chatwoot pede versões específicas e um ecossistema típico de Linux, o que costuma ser mais direto de instalar e manter em uma distro Ubuntu no WSL do que tentar reproduzir tudo nativamente no Windows.

Além disso, a documentação oficial tem guias de desenvolvimento focados em Ubuntu e também em Docker/Docker Compose, que encaixam naturalmente em um ambiente Linux.

## Docker e fluxo de dev

O fluxo de desenvolvimento recomendado pelo Chatwoot com Docker usa comandos como `docker compose up`, `docker compose exec`, migrations, logs etc., e isso roda de forma bem previsível quando o projeto e o Docker estão no "mundo Linux".

No Windows, dá para fazer também, mas o atrito aumenta quando a toolchain e o filesystem estão misturados entre Windows e Linux.

> Este guia cobre o setup ponta-a-ponta: pacotes do SO, linguagens, serviços, `.env`, migrações e comandos de start, com um catálogo de falhas e correções.

## Problemas comuns com CRLF (Windows vs. Linux)

- Git mostra muitos arquivos como modificados: no WSL, arquivos criados/convertidos no Windows podem ficar com CRLF, enquanto Linux/WSL espera LF. Isso gera diffs falsos e avisos como "LF will be replaced by CRLF".
- Erros em builds Docker/Ruby/Node: scripts shell, `Procfile.dev`, YAML ou configs com CRLF podem quebrar em ambientes Linux (ex.: "command not found" ou erro de parse).
- Solução rápida:
  - No Git do WSL/Linux: `git config --global core.autocrlf input`
  - No Git do Windows: `git config --global core.autocrlf true`
  - Preferir clonar e trabalhar dentro do filesystem do WSL (ex.: `~/...`), evitando `/mnt/c/...` quando possível.
  - Usar `.gitattributes` para forçar `eol=lf` em arquivos sensíveis.

---

## Sumário

1. [Stack, Tecnologias e Dependências](#stack-tecnologias-e-dependências)
2. [Preparação do sistema](#preparação-do-sistema)
   - 2.1. [WSL e VS Code](#wsl-e-vs-code)
   - 2.2. [Pacotes essenciais do SO](#pacotes-essenciais-do-so)
3. [Linguagens & Toolchain](#linguagens--toolchain)
   - 3.1. [Ruby via RVM (3.4.4)](#ruby-via-rvm-344)
   - 3.2. [Bundler (2.5.16)](#bundler-2516)
   - 3.3. [Node & pnpm](#node--pnpm)
4. [Serviços](#serviços)
   - 4.1. [PostgreSQL + pgvector](#postgresql--pgvector)
   - 4.2. [Redis](#redis)
5. [Setup do projeto](#setup-do-projeto)
   - 5.1. [Clonar / entrar no repo](#clonar--entrar-no-repo)
   - 5.2. [`.env` (Desenvolvimento)](#env-desenvolvimento)
   - 5.3. [Instalar dependências](#instalar-dependências)
   - 5.4. [Preparar DB & seed](#preparar-db--seed)
   - 5.5. [Rodar a aplicação](#rodar-a-aplicação)
6. [Comandos de Start (TL;DR)](#comandos-de-start-tldr)
7. [Alvos `make` (cheat sheet)](#alvos-make-cheat-sheet)
8. [Pré-check (Validar ambiente)](#pré-check-validar-ambiente)
9. [Checklist rápido](#checklist-rápido)
10. [Login padrão (DEV)](#login-padrão-dev)
11. [Catálogo de falhas (erros & correções)](#catálogo-de-falhas-erros--correções)
12. [Comandos de diagnóstico](#comandos-de-diagnóstico)
13. [Apêndice: `.env` completo (DEV)](#apêndice-env-completo-dev)

---

## Stack, Tecnologias e Dependências

- Ruby **3.4.4** via **RVM** + Bundler **2.5.16**
- Node **20.x** (recomendado) ou **23.x** + **pnpm**
- PostgreSQL local (com opção de **pgvector**)
- Redis local
- Dependências (`bundle install`, `pnpm install`)
- `.env` por projeto
- `rails db:chatwoot_prepare` + (opcional) `db:seed`
- Start via Foreman (`Procfile.dev`) ou Overmind

Observação sobre versões:
- Antes de começar, valide se o `Gemfile.lock` do seu fork realmente usa Ruby 3.4.4. Se for diferente (ex.: 3.3.3), ajuste os comandos correspondentemente.

---

## Preparação do sistema

### WSL e VS Code

1. Instale WSL2 e a distro Ubuntu-24.04.
2. No VS Code, instale a extensão Remote – WSL.
3. Abra o repo do lado do WSL:
   ```bash
   cd ~/Impulso_Hub && code .
   ```
4. Para evitar problema de fim de linha (CRLF):
   - VS Code (workspace): `files.eol = "\n"`
   - Git no WSL/Linux: `git config --global core.autocrlf input`

#### `/etc/wsl.conf` — por que setamos isso?

Edite (como root) o arquivo `/etc/wsl.conf` no Ubuntu (WSL) e adicione:

```ini
[interop]
enabled = true
appendWindowsPath = true

[boot]
systemd = true
```

O que isso faz:
- `[interop] enabled = true` permite que o WSL execute binários do Windows (ex.: `powershell.exe`).
- `appendWindowsPath = true` adiciona as pastas do Windows no `PATH` do WSL (ex.: `C:\Windows\System32`).
- `[boot] systemd = true` habilita o systemd no WSL, necessário para `systemctl` funcionar (ex.: iniciar `postgresql` e `redis-server`).

Nota sobre VS Code:
- A forma recomendada é usar Remote – WSL, que executa o servidor do VS Code dentro do WSL, evitando inconsistências de toolchain.

Após alterar o arquivo, reinicie o WSL no Windows:

```powershell
wsl --shutdown
```

Abra novamente o Ubuntu-24.04.

### Pacotes essenciais do SO

```bash
sudo apt-get update
sudo apt-get install -y \
  git curl build-essential software-properties-common \
  postgresql postgresql-contrib libpq-dev \
  redis-server imagemagick tmux dos2unix
```

Por quê:
- `build-essential` compila gems nativas.
- `libpq-dev` permite compilar o adapter do Postgres.
- `imagemagick` pode ser usado pelo Rails/ActiveStorage.
- `dos2unix` corrige finais de linha do Windows.
- `tmux` é necessário para Overmind.

---

## Linguagens & Toolchain

### Ruby via RVM (3.4.4)

```bash
# carregar RVM (se já instalado)
source /etc/profile.d/rvm.sh 2>/dev/null || source ~/.rvm/scripts/rvm

# instalar/selecionar Ruby 3.4.4
rvm install 3.4.4
rvm use 3.4.4 --default

ruby -v
```

### Bundler (2.5.16)

```bash
gem install bundler -v 2.5.16
bundle -v
```

### Node & pnpm

```bash
# Node 20 (recomendado)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v

# pnpm via corepack
corepack enable
corepack prepare pnpm@10 --activate
pnpm -v
```

---

## Serviços

### PostgreSQL + pgvector

Criar usuário/DB de dev (idempotente):

```bash
sudo systemctl enable --now postgresql

sudo -u postgres psql -c "DO $$BEGIN IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='dev') THEN CREATE ROLE dev LOGIN SUPERUSER PASSWORD 'devpass'; END IF; END$$;"
sudo -u postgres createdb -O dev chatwoot_development 2>/dev/null || true

# teste
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "select current_user, current_database();"
```

Instalar pgvector (se as migrações pedirem):

```bash
# Detectar versão major do PostgreSQL instalada
PG_VERSION=$(psql --version | awk '{print $NF}' | cut -d. -f1)
echo "PostgreSQL version: $PG_VERSION"

# Instalar pgvector para essa versão (fallback para 16/15, se necessário)
sudo apt-get install -y postgresql-$PG_VERSION-pgvector || \
  (echo "pgvector não disponível para PG $PG_VERSION; tentando versões próximas..." && \
   sudo apt-get install -y postgresql-16-pgvector 2>/dev/null || \
   sudo apt-get install -y postgresql-15-pgvector)

# Criar extensão
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Redis

```bash
sudo systemctl enable --now redis-server
redis-cli ping   # espera PONG
```

---

## Setup do projeto

### Clonar / entrar no repo

```bash
cd ~
git clone https://github.com/ImpulsoCoreIA/Impulso_Hub.git
cd ~/Impulso_Hub
```

### `.env` (Desenvolvimento)

Crie `~/Impulso_Hub/.env` com finais de linha LF (sem BOM):

```dotenv
RAILS_ENV=development
FRONTEND_URL=http://localhost:3000
SECRET_KEY_BASE=SUBSTITUA_POR_HEX_128_CHARS

# Postgres
POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_development
POSTGRES_USERNAME=dev
POSTGRES_PASSWORD=devpass

# Redis
REDIS_URL=redis://127.0.0.1:6379

# Email (DEV)
MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"
LETTER_OPENER=true

# Úteis no DEV
FORCE_SSL=false
ENABLE_ACCOUNT_SIGNUP=true
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=debug
LOG_SIZE=500
```

Gerar um secret forte (rode na WSL com Ruby já instalado):

```bash
ruby -e 'require "securerandom"; puts SecureRandom.hex(64)'
```

Se criou o arquivo no Windows, normalize:

```bash
dos2unix .env || sed -i 's/\r$//' .env
sed -i '1s/^\xEF\xBB\xBF//' .env   # remove BOM, se houver
chmod 600 .env
```

#### Validar `.env`

```bash
for var in RAILS_ENV POSTGRES_HOST POSTGRES_USERNAME REDIS_URL SECRET_KEY_BASE; do
  if grep -q "^$var=" .env; then
    echo "OK: $var"
  else
    echo "FALTANDO: $var"
  fi
done
```

### Instalar dependências

```bash
bundle install
pnpm install
# ou: make burn
```

### Preparar DB & seed

```bash
RAILS_ENV=development bundle exec rails db:chatwoot_prepare
# (opcional) dados de exemplo:
RAILS_ENV=development bundle exec rails db:seed
```

Se o Rails não estiver lendo o `.env`, force via `DATABASE_URL`:

```bash
RAILS_ENV=development \
DATABASE_URL=postgres://dev:devpass@127.0.0.1:5432/chatwoot_development \
bundle exec rails db:chatwoot_prepare
```

Aguardar PostgreSQL com retry (quando necessário):

```bash
for i in {1..10}; do
  PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "select 1;" && break
  echo "Tentativa $i/10... aguardando PostgreSQL"
  sleep 2
done
```

### Rodar a aplicação

- Foreman:
  ```bash
  gem install foreman
  foreman start -f Procfile.dev
  ```
  Acesse: http://localhost:3000

- Overmind (requer binário + `tmux`):
  ```bash
  sudo apt-get install -y tmux

  # Instale o binário do Overmind (ex.: em /usr/local/bin)
  # Releases: https://github.com/DarthSim/overmind/releases

  overmind s -f Procfile.dev
  # ou, se o Makefile tiver:
  make run        # start
  make force_run  # ignora socket/zombie
  ```

- Smoke test (backend puro):
  ```bash
  RAILS_ENV=development bundle exec rails s -p 3000 -b 0.0.0.0
  ```
  Em outro terminal:
  ```bash
  pnpm run dev   # ou: pnpm vite
  ```

---

## Comandos de Start (TL;DR)

Escolha um fluxo:

```bash
# Foreman (recomendado, mais simples)
gem install foreman
foreman start -f Procfile.dev

# Overmind (mais robusto)
sudo apt-get install -y tmux
# Instalar overmind: https://github.com/DarthSim/overmind/releases
overmind s -f Procfile.dev

# Rails + Vite (separados, para debug)
RAILS_ENV=development bundle exec rails s -p 3000 -b 0.0.0.0
# em outro terminal
pnpm run dev
```

---

## Alvos `make` (cheat sheet)

Dependem do `Makefile` do repo:

- `make burn` → `bundle install && pnpm install`
- `make db` → `rails db:chatwoot_prepare`
- `make db_seed` → `rails db:seed`
- `make run` / `make force_run` → iniciar processos (geralmente via Overmind)

Se aparecer "No rule to make target", verifique o `Makefile` do repo.

---

## Pré-check (Validar ambiente)

Salve como `precheck.sh` e rode `bash precheck.sh`:

```bash
#!/bin/bash
set +e

echo "=== PRE-CHECK IMPULSO_HUB DEV ==="
echo ""

echo -n "Ruby (esperado 3.4.4): "
ruby -v | grep -q 3.4.4 && echo "OK" || echo "DIVERGENTE"

echo -n "Bundler (esperado 2.5.16): "
bundle -v | grep -q 2.5.16 && echo "OK" || echo "DIVERGENTE"

echo -n "Node (recomendado 20.x): "
node -v | grep -q 'v20\.' && echo "OK" || echo "DIVERGENTE"

echo -n "pnpm instalado: "
command -v pnpm &>/dev/null && echo "OK" || echo "AUSENTE"

echo -n "PostgreSQL rodando: "
sudo systemctl is-active --quiet postgresql && echo "OK" || echo "PARADO"

echo -n "Redis rodando: "
sudo systemctl is-active --quiet redis-server && echo "OK" || echo "PARADO"

echo -n ".env existe: "
test -f .env && echo "OK" || echo "AUSENTE"

if test -f .env; then
  echo ""
  echo "Validando .env:"
  for var in RAILS_ENV POSTGRES_HOST POSTGRES_USERNAME REDIS_URL SECRET_KEY_BASE; do
    echo -n "  $var: "
    grep -q "^$var=" .env && echo "OK" || echo "AUSENTE"
  done
fi

echo ""
echo "=== Fim do pre-check ==="
```

---

## Checklist rápido

1. `rvm use 3.4.4 --default` e `ruby -v` confere com a versão do projeto.
2. `bundle -v` confere com a versão do `Gemfile.lock`.
3. `sudo systemctl enable --now postgresql redis-server`.
4. Usuário/DB ok: `select 1;` retorna `1`.
5. `.env` existe em `~/Impulso_Hub` e está em LF (sem BOM).
6. `bundle install && pnpm install`.
7. `rails db:chatwoot_prepare` (ou com `DATABASE_URL`).
8. `foreman start -f Procfile.dev` ou `overmind s -f Procfile.dev`.
9. Acessar `http://localhost:3000`.
10. Confirmar login com credenciais padrão ou criar via seed/UI.

---

## Login padrão (DEV)

- Email: `john@acme.inc`
- Senha: `Password1!`

Se não existir ou não funcionar:

```bash
RAILS_ENV=development bundle exec rails db:seed
```

Observação:
- Valide se essas credenciais permanecem as mesmas no fork (podem ter sido alteradas).

---

## Catálogo de falhas (erros & correções)

### Ruby/Bundler

**`bundle: not found`**
- RVM não carregou ou bundler não instalado:
  ```bash
  source /etc/profile.d/rvm.sh 2>/dev/null || source ~/.rvm/scripts/rvm
  rvm use 3.4.4 --default
  gem install bundler -v 2.5.16
  which bundle && bundle -v
  ```

**`Your Ruby version is 3.3.3, but your Gemfile specified 3.4.4`**
- Use a Ruby correta:
  ```bash
  rvm install 3.4.4 && rvm use 3.4.4 --default
  bundle install
  ```

**`bundler: command not found: rails`**
- `bundle install` não terminou ou está com Ruby errada:
  ```bash
  ruby -e 'puts Gem.bindir'
  echo $PATH
  ```

### Node/pnpm/Vite

**Aviso de engines (pede Node 23.x)**
- Node 20 costuma funcionar em desenvolvimento; use 23 se o projeto exigir.

**Browserslist desatualizado**
```bash
pnpm dlx update-browserslist-db@latest
```

**Vite CJS deprecation**
- Informativo; geralmente pode ser ignorado em DEV.

### PostgreSQL/pgvector

**`fe_sendauth: no password supplied`**
- `.env` ausente, no caminho errado ou com CRLF/BOM:
  ```bash
  set -a; source .env; set +a
  PGPASSWORD="$POSTGRES_PASSWORD" psql -h "$POSTGRES_HOST" -U "$POSTGRES_USERNAME" -d "$POSTGRES_DATABASE" -c "select 1;"
  ```
- Forçar migrações:
  ```bash
  RAILS_ENV=development DATABASE_URL=postgres://dev:devpass@127.0.0.1:5432/chatwoot_development \
  bundle exec rails db:chatwoot_prepare
  ```

**Postgres não está rodando**
```bash
sudo systemctl enable --now postgresql
ss -lntp | grep 5432
```

**`localhost` vs `127.0.0.1`**
- Se houver inconsistência de resolução, use `127.0.0.1` no `.env`:
  ```dotenv
  POSTGRES_HOST=127.0.0.1
  ```

### Redis

**Não conecta / parado**
```bash
sudo systemctl enable --now redis-server
redis-cli ping
```

### `.env` / dotenv

**`dotenv: No such file or directory @ rb_sysopen - /home/.../.env`**
- Garanta que o arquivo existe em `~/Impulso_Hub/.env` e normalize EOL:
  ```bash
  dos2unix .env || sed -i 's/\r$//' .env
  sed -i '1s/^\xEF\xBB\xBF//' .env
  chmod 600 .env
  ```

**Valores com espaço (ex.: `MAILER_SENDER_EMAIL`)**
- Sempre usar aspas:
  ```dotenv
  MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"
  ```

### WSL / VS Code / Interop

**`Exec format error` ao chamar `code .` / `.exe`**
- Verifique `wsl.conf` (interop e systemd) e use Remote – WSL.
- Reinicie WSL: `wsl --shutdown` no Windows (PowerShell).

**Arquivo criado no Windows com CRLF**
- Converta:
  ```bash
  dos2unix .env
  ```

**Script falha com "command not found" (suspeita de CRLF)**
```bash
file Procfile.dev
dos2unix Procfile.dev
```

### Miscelânea (portas, seeds, assets)

**Porta 3000 em uso**
```bash
ss -lntp | grep :3000
kill -9 <PID>
```

**Login padrão ausente ou credenciais inválidas**
```bash
RAILS_ENV=development bundle exec rails db:seed
```

Se precisar criar manualmente:
```bash
RAILS_ENV=development bundle exec rails c
# Exemplos genéricos; ajuste conforme o modelo do fork:
# Account.create_with_defaults(name: "Acme Corp")
# User.create(email: "john@acme.inc", password: "Password1!", account_id: Account.last.id)
```

**Problemas de assets**
```bash
bundle exec rails assets:precompile
```

---

## Comandos de diagnóstico

```bash
# Ruby/Bundler
which ruby; ruby -v
which bundle; bundle -v
ruby -e 'puts Gem.bindir'
echo $PATH

# Node/pnpm
node -v; pnpm -v

# Repo & env
pwd; ls -l .env
grep -nE 'RAILS_ENV|FRONTEND_URL|POSTGRES_|REDIS_URL|SECRET_KEY_BASE' .env || true

# Postgres
sudo systemctl status postgresql --no-pager
PGPASSWORD=devpass psql -h 127.0.0.1 -U dev -d chatwoot_development -c "select 1;"
ss -lntp | grep 5432

# Redis
sudo systemctl status redis-server --no-pager
redis-cli ping

# App
RAILS_ENV=development bundle exec rails -v
```

No Windows (PowerShell), para conferir distros WSL:
```powershell
wsl -l -v
```

---

## Apêndice: `.env` completo (DEV)

```dotenv
RAILS_ENV=development
FRONTEND_URL=http://localhost:3000
SECRET_KEY_BASE=SUBSTITUA_POR_HEX_128_CHARS

POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_development
POSTGRES_USERNAME=dev
POSTGRES_PASSWORD=devpass

REDIS_URL=redis://127.0.0.1:6379

MAILER_SENDER_EMAIL="Impulso Hub <no-reply@localhost>"
LETTER_OPENER=true

FORCE_SSL=false
ENABLE_ACCOUNT_SIGNUP=true
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=debug
LOG_SIZE=500
```

---

## Notas finais

- Mantenha este documento atualizado quando versões forem alteradas (Ruby, Node, PostgreSQL, etc.).
- Teste em uma WSL2 limpa periodicamente para garantir que o fluxo ainda funciona.
- Priorize clonar e trabalhar dentro do filesystem do WSL (ex.: `~/...`) para reduzir problemas de performance, permissões e CRLF.
- Use `systemctl` para gerenciar serviços; vários fluxos dependem de systemd estar habilitado.

---

Versão: 1.0
Data: 2026-01-17
Testado em: WSL2 Ubuntu 24.04, Ruby 3.4.4, Node 20.x
