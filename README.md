# Manual de Instalação e Configuração — LiteLLM Gateway (Docker no WSL2) + LM Studio + Claude Code

**Escopo:** Gateway LLM compartilhado em rede local (sem exposição à internet), com modelos Anthropic, Gemini e modelos open source locais. **Estratégia:** LiteLLM e PostgreSQL rodam em containers Docker dentro do WSL2 do PC Servidor. LM Studio permanece nativo no Windows host (é GUI). Cliente: Claude Code CLI com suporte nativo a gateway via `ANTHROPIC_BASE_URL`.

> 📌 **Comparação com a versão anterior (NSSM nativo):** veja `README.md`. Esta versão substitui a instalação de Python + LiteLLM + Prisma + PostgreSQL + NSSM por dois containers gerenciados via `docker compose`. O ganho operacional é grande: nenhum patch de SSL/Unicode em arquivos do site-packages, nenhuma briga com o `query-engine` do Prisma sendo morto pelo McAfee no host, atualização do gateway por troca de imagem, rollback por reversão do volume.

---

## Sumário

1. [Visão geral da arquitetura](#1-visão-geral-da-arquitetura)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Parte 1 — Configuração do PC Servidor](#parte-1--configuração-do-pc-servidor-1015006)
   - 3.1 [Habilitar WSL2 e instalar Ubuntu](#31-habilitar-wsl2-e-instalar-ubuntu)
   - 3.2 [Configurar `.wslconfig` (networking mirrored + systemd)](#32-configurar-wslconfig-networking-mirrored--systemd)
   - 3.3 [Instalar Docker Engine dentro do WSL2](#33-instalar-docker-engine-dentro-do-wsl2)
   - 3.4 [Instalação e configuração do LM Studio (no Windows host)](#34-instalação-e-configuração-do-lm-studio-no-windows-host)
   - 3.5 [Download dos modelos open source](#35-download-dos-modelos-open-source)
   - 3.6 [Servidor do LM Studio acessível ao WSL2](#36-servidor-do-lm-studio-acessível-ao-wsl2)
   - 3.7 [Preparar a pasta do projeto e os arquivos do compose](#37-preparar-a-pasta-do-projeto-e-os-arquivos-do-compose)
   - 3.8 [Geração da master key segura](#38-geração-da-master-key-segura)
   - 3.9 [Preencher o `.env`](#39-preencher-o-env)
   - 3.10 [Subir os containers](#310-subir-os-containers)
   - 3.11 [Firewall do Windows (LAN → porta 4000)](#311-firewall-do-windows-lan--porta-4000)
   - 3.12 [Geração de virtual keys individuais para devs](#312-geração-de-virtual-keys-individuais-para-devs)
   - 3.13 [Acesso e uso do dashboard do LiteLLM](#313-acesso-e-uso-do-dashboard-do-litellm)
4. [Parte 2 — Configuração do Notebook do Dev](#parte-2--configuração-do-notebook-do-dev)
   - 4.1 [Instalação do Node.js](#41-instalação-do-nodejs)
   - 4.2 [Instalação do Claude Code CLI](#42-instalação-do-claude-code-cli)
   - 4.3 [Configuração das variáveis de ambiente do gateway](#43-configuração-das-variáveis-de-ambiente-do-gateway)
   - 4.4 [Configuração persistente no perfil do PowerShell](#44-configuração-persistente-no-perfil-do-powershell)
   - 4.5 [Instalação e configuração do OpenSpec](#45-instalação-e-configuração-do-openspec)
   - 4.6 [Teste de conexão](#46-teste-de-conexão)
   - 4.7 [Uso prático no dia a dia](#47-uso-prático-no-dia-a-dia)
5. [Troubleshooting](#5-troubleshooting)
6. [Comandos de manutenção](#6-comandos-de-manutenção)
7. [Checklist final de segurança](#7-checklist-final-de-segurança)

---

## 1. Visão geral da arquitetura

```
┌─ Rede local 10.150.0.0/24 ─────────────────────────────────────────────┐
│                                                                         │
│   Notebook Dev 1                    Notebook Dev 2                      │
│   Claude Code CLI + OpenSpec        Claude Code CLI + OpenSpec          │
│   ANTHROPIC_BASE_URL=:4000          ANTHROPIC_BASE_URL=:4000            │
│   ANTHROPIC_AUTH_TOKEN=sk-vkey-1    ANTHROPIC_AUTH_TOKEN=sk-vkey-2      │
│         │                                    │                          │
│         └──────────────┬─────────────────────┘                          │
│                        │ HTTP :4000                                     │
│                        ▼                                                │
│        ┌─────────────────────────────────────────────────┐              │
│        │  PC Servidor — 10.150.0.69 (Windows 11)         │              │
│        │                                                 │              │
│        │   ┌── LM Studio :1234 ──┐                       │              │
│        │   │ (Windows host)      │                       │              │
│        │   │ deepseek / qwen     │                       │              │
│        │   └────────┬────────────┘                       │              │
│        │            ▲ host.docker.internal:1234          │              │
│        │            │                                    │              │
│        │   ┌────────┴───── WSL2 (Ubuntu) ─────────────┐  │              │
│        │   │  Docker Engine                           │  │              │
│        │   │                                          │  │              │
│        │   │   ┌──────────────────────┐               │  │              │
│        │   │   │ litellm-gateway      │ :4000 (host)  │  │              │
│        │   │   │ ghcr.io/berriai/...  │◄──┐           │  │              │
│        │   │   └──────────┬───────────┘   │ network   │  │              │
│        │   │              │ DATABASE_URL  │ default   │  │              │
│        │   │   ┌──────────▼───────────┐   │           │  │              │
│        │   │   │ litellm-postgres     │◄──┘           │  │              │
│        │   │   │ postgres:16-alpine   │  vol named    │  │              │
│        │   │   └──────────────────────┘  postgres_dt  │  │              │
│        │   └──────────────────────────────────────────┘  │              │
│        └─────────────────────────────────────────────────┘              │
└─────────────────────────┼───────────────────────────────────────────────┘
                          │ HTTPS
                          ▼
              ┌──────────────────────────┐
              │ APIs externas            │
              │ • Anthropic              │
              │ • Google Gemini          │
              └──────────────────────────┘
```

**Stack completa por camada:**

| Camada | Componente | Onde roda |
|---|---|---|
| **Cliente** | Claude Code CLI + OpenSpec | Notebook do dev |
| **Gateway** | LiteLLM (container) | WSL2 Ubuntu / Docker Engine — `litellm-gateway` |
| **Banco** | PostgreSQL 16 (container) | WSL2 Ubuntu / Docker Engine — `litellm-postgres` |
| **Runtime local** | LM Studio | Windows host :1234 |
| **APIs externas** | Anthropic, Gemini | Internet (HTTPS) |

**Princípios de segurança aplicados:**

- API keys reais (Anthropic, Gemini) ficam **apenas no `.env` do PC Servidor**, lidas pelo container via `environment` — nunca distribuídas
- Cada dev recebe uma **virtual key individual** com permissões e budget próprios
- Claude Code usa `ANTHROPIC_AUTH_TOKEN` (virtual key) — **nunca** a API key real da Anthropic
- Porta 4000 acessível **apenas na LAN** (firewall do Windows restringe ao range `10.150.0.0/24`)
- **Sem exposição à internet** — sem port forwarding no roteador
- LM Studio escuta na rede do WSL apenas (firewall do Windows bloqueia a porta 1234 para a LAN externa)
- Postgres do gateway **não publica porta** — só fala com o LiteLLM pela rede interna do compose

---

## 2. Pré-requisitos

**No PC Servidor (10.150.0.69):**

- Windows 11 Pro **22H2 ou superior** (necessário para `networkingMode=mirrored` do WSL2)
- Acesso administrativo (PowerShell elevado, firewall, instalação de programas)
- Conexão com a internet (para `wsl --install`, `docker pull`, e chamadas às APIs Anthropic/Gemini)
- Mínimo 16 GB de RAM, recomendado 24 GB+ (modelos open source + containers + LM Studio)
- ~30 GB livres em disco (modelos quantizados ocupam de 6 a 13 GB cada; imagens Docker ~1 GB)
- IP fixo `10.150.0.69` configurado na placa de rede
- Virtualização habilitada na BIOS (Intel VT-x ou AMD-V) — sem isso o WSL2 não inicia

**Nos notebooks dos devs:**

- Windows, macOS ou Linux
- Acesso à mesma rede local (range `10.150.0.0/24`)
- Node.js 20.19.0 ou superior (requisito do Claude Code e do OpenSpec CLI)
- Permissão para ler/escrever em `~/.claude/`

---

# Parte 1 — Configuração do PC Servidor (10.150.0.69)

## 3.1 Habilitar WSL2 e instalar Ubuntu

**PowerShell como administrador:**

```powershell
# Instala WSL2 + distro padrão (Ubuntu) em um comando só
wsl --install
```

Reinicie o Windows quando solicitado. Na primeira inicialização do Ubuntu o sistema vai pedir um usuário e senha — anote a senha (será usada para `sudo` dentro do WSL).

Verificar:

```powershell
wsl --status
wsl --list --verbose
```

A saída deve mostrar `Ubuntu` com `VERSION 2`. Se vier como `VERSION 1`, converta:

```powershell
wsl --set-version Ubuntu 2
wsl --set-default-version 2
```

## 3.2 Configurar `.wslconfig` (networking mirrored + systemd)

O modo **mirrored networking** (Windows 11 22H2+) faz o WSL2 compartilhar a interface de rede do Windows — assim qualquer porta exposta no WSL2 (ex: `4000`) fica acessível diretamente na LAN sem `netsh portproxy`.

**Criar `C:\Users\<seu-usuario>\.wslconfig`** com o conteúdo:

```ini
[wsl2]
networkingMode=mirrored
firewall=true
dnsTunneling=true
autoProxy=true

[experimental]
hostAddressLoopback=true
```

> 📝 O bloco `[experimental]` permite que serviços em `localhost` do Windows host (caso do LM Studio em `127.0.0.1:1234`) sejam acessíveis a partir do WSL2. Sem isso seria necessário expor o LM Studio em `0.0.0.0` e abrir firewall — a alternativa funciona, mas é menos limpa.

**Habilitar `systemd` dentro do Ubuntu** (necessário para o Docker iniciar como serviço gerenciado):

```powershell
wsl -d Ubuntu -- sudo tee /etc/wsl.conf <<'EOF'
[boot]
systemd=true

[network]
generateResolvConf=true
EOF
```

**Reiniciar o WSL para aplicar tudo:**

```powershell
wsl --shutdown
# aguarde 8 segundos antes de reabrir o WSL
wsl
```

Dentro do WSL, confirme:

```bash
# systemd ativo?
systemctl is-system-running
# Esperado: running  (ou degraded — também ok)

# Endereço de rede do WSL = mesma sub-rede do Windows host?
ip addr show eth0
```

## 3.3 Instalar Docker Engine dentro do WSL2

> ⚠️ **Não use Docker Desktop.** Você escolheu Docker Engine puro dentro do WSL2 — instalação mais leve, sem licença comercial, sem dependência do daemon do Desktop no Windows.

**Dentro do Ubuntu (WSL):**

```bash
# 1. Remover qualquer instalação antiga
sudo apt-get remove -y docker docker-engine docker.io containerd runc

# 2. Pré-requisitos
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# 3. Adicionar a chave GPG oficial do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4. Adicionar o repositório oficial
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Instalar Docker Engine + Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# 6. Adicionar seu usuário ao grupo docker (evita sudo a cada comando)
sudo usermod -aG docker $USER

# 7. Habilitar o serviço para subir junto com o systemd do WSL
sudo systemctl enable --now docker
```

**Saia e reentre no WSL** para que a membership no grupo `docker` tenha efeito:

```bash
exit
```

```powershell
wsl --shutdown
wsl
```

**Validação:**

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Se `hello-world` imprime a mensagem de sucesso, o Docker está pronto.

### Rede corporativa com proxy SSL auto-assinado (opcional)

Se o `docker pull` falhar com erro de certificado:

```bash
# Adicionar o CA do proxy corporativo ao trust store do Ubuntu
sudo cp /mnt/c/caminho/para/proxy-ca.crt /usr/local/share/ca-certificates/proxy-ca.crt
sudo update-ca-certificates

# Reiniciar o daemon
sudo systemctl restart docker
```

## 3.4 Instalação e configuração do LM Studio (no Windows host)

O LM Studio continua sendo instalado nativamente no Windows porque é GUI e gerencia GPU/CPU nativamente. Os containers do WSL2 acessam o LM Studio via `host.docker.internal`.

1. Acesse https://lmstudio.ai/ e baixe a versão mais recente para Windows
2. Execute o instalador como administrador
3. Aceite as configurações padrão durante a instalação
4. Após a primeira execução, vá em **Settings → General**:
   - Marque **"Run LM Studio at startup"**
   - Marque **"Start minimized to system tray"**
5. Vá em **Settings → Developer**:
   - Marque **"Auto-start LLM server on app launch"**
   - **"Server port"** = `1234`
   - **"Serve on local network"** = **OFF** (manter restrito ao localhost do Windows)

> 💡 **Por que LM Studio em localhost se o container precisa acessar?** Graças ao `hostAddressLoopback=true` no `.wslconfig` (seção 3.2), o WSL2 enxerga o `127.0.0.1` do Windows host. Combinado com `extra_hosts: host.docker.internal:host-gateway` no compose, o container resolve `host.docker.internal` → IP da rede vEthernet (WSL) → loopback do Windows. Não é necessário expor o LM Studio na LAN.

## 3.5 Download dos modelos open source

Na aba **Discover** (lupa) do LM Studio, baixe os três modelos quantizados:

| Modelo | Quantização | Tamanho | Identificador para busca |
|---|---|---|---|
| DeepSeek-Coder-V2-Lite | Q4_K_M | ~10 GB | `deepseek-coder-v2-lite-instruct GGUF` (versão lmstudio-community) |
| Qwen2.5-Coder 14B | Q4_K_M | ~9 GB | `qwen2.5-coder-14b-instruct GGUF` (versão lmstudio-community) |
| Qwen2.5-Coder 7B | Q6_K | ~6 GB | `qwen2.5-coder-7b-instruct GGUF` (versão lmstudio-community) |

Para cada modelo:

1. Pesquise pelo nome
2. Selecione o arquivo com a quantização correta
3. Clique em **Download**
4. Anote o **identificador exato** que aparece no LM Studio após o download — necessário no `config.docker.yaml`

> 💡 Para descobrir o identificador exato, vá na aba **My Models** do LM Studio. O nome listado ali é o que o servidor expõe na API.

## 3.6 Servidor do LM Studio acessível ao WSL2

1. No LM Studio, vá na aba **Developer** (ícone `</>`)
2. Clique em **Start Server** (deve ficar verde, indicando "Running on port 1234")
3. Em **Server Options**:
   - **Server Port:** `1234`
   - **JIT (Just-In-Time) Loading:** habilitado
   - **Auto-Unload:** habilitado (descarrega após 10 minutos de inatividade)
   - **CORS:** desabilitado

**Verificar acesso a partir do WSL** (com o LM Studio rodando):

```bash
# Dentro do WSL Ubuntu
curl http://host.docker.internal:1234/v1/models
```

Deve retornar JSON com a lista de modelos baixados. Se der erro de conexão, confirme em `.wslconfig` que `[experimental] hostAddressLoopback=true` está presente e que o WSL foi reiniciado (`wsl --shutdown` + reabrir).

## 3.7 Preparar a pasta do projeto e os arquivos do compose

A pasta `C:\litellm` é montada automaticamente dentro do WSL como `/mnt/c/litellm`. Vamos trabalhar a partir dela (assim você edita arquivos com VS Code/Notepad no Windows e o container vê tudo).

> ⚠️ **Performance:** rodar containers Docker com volumes montados em `/mnt/c/...` é mais lento que mover o projeto para o filesystem nativo do WSL (ex: `~/litellm`). Para este caso (config + um YAML), a diferença é irrelevante. Para projetos maiores, mova para o home do WSL.

**No WSL, navegue até o projeto:**

```bash
cd /mnt/c/litellm
ls -la
```

Os três arquivos necessários já existem no repositório:

| Arquivo | Função |
|---|---|
| `docker-compose.yml` | Definição dos serviços `postgres` e `litellm` |
| `config.docker.yaml` | Configuração do LiteLLM (modelos, fallbacks, routing) com `api_base` para LM Studio em `host.docker.internal:1234` |
| `.env.example` | Template das variáveis (Anthropic, Gemini, master key, credenciais Postgres) |

> 📝 O `docker-compose.yml` neste repositório monta `./config.yaml`. Para usar a versão Docker, ajuste a linha de `volumes` para apontar para `./config.docker.yaml` OU renomeie o arquivo. **Opção recomendada (mais limpa):**
>
> ```bash
> # Renomear (no WSL ou no Windows)
> cp config.docker.yaml config.yaml.docker-backup
> ```
>
> ou edite `docker-compose.yml` para refletir o nome real.

## 3.8 Geração da master key segura

A master key é a credencial **administrativa** do gateway. Permite gerar virtual keys, acessar o dashboard administrativo e fazer qualquer operação. **Nunca** distribua para os devs.

**No WSL Ubuntu (recomendado — usa `/dev/urandom`):**

```bash
echo "sk-$(openssl rand -base64 48 | tr '+/' '-_' | tr -d '=' | head -c 64)"
```

**Alternativa via PowerShell no Windows:**

```powershell
$bytes = New-Object byte[] 48
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
"sk-" + [Convert]::ToBase64String($bytes).Replace("+","-").Replace("/","_").Replace("=","")
```

Saída exemplo:

```
sk-aB3xZ9-Kp_qN7vR4tY2mFw8GhJlE6sCnD0bV5oXuI1zPyT-rQmHaSwLkUvN
```

**Boas práticas:**

- Anote em um gerenciador de senhas (Bitwarden, 1Password, KeePass)
- Nunca commite em repositórios Git
- Nunca envie por WhatsApp/email sem criptografia

Vamos chamá-la daqui em diante de `<MASTER_KEY>`.

## 3.9 Preencher o `.env`

```bash
cd /mnt/c/litellm
cp .env.example .env
```

**Abra `.env`** (em Windows: `C:\litellm\.env`; em WSL: `nano .env`) e preencha:

```env
# ---- API keys reais ---------------------------------
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxx
GEMINI_API_KEY=AIzaSyxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ---- Credencial administrativa do gateway -----------
LITELLM_MASTER_KEY=sk-suaMasterKeyGeradaNa3.8...

# ---- PostgreSQL -------------------------------------
POSTGRES_USER=litellm_user
POSTGRES_PASSWORD=trocar-por-senha-forte-aqui
POSTGRES_DB=litellm
```

**Confirme que `.env` está ignorado pelo Git** (já no `.gitignore` do projeto):

```bash
git check-ignore -v .env
# Esperado: .gitignore:N:.env  .env
```

## 3.10 Subir os containers

Ainda no WSL, dentro de `/mnt/c/litellm`:

```bash
# Baixar imagens e subir em background
docker compose up -d

# Acompanhar logs durante a inicialização
docker compose logs -f
```

Você verá:
- `litellm-postgres` aplicando migrations automáticas (logs de `prisma migrate deploy`)
- `litellm-gateway` iniciando o servidor Uvicorn em `0.0.0.0:4000`
- Logs do tipo `INFO: Uvicorn running on http://0.0.0.0:4000`

Encerre o follow com `Ctrl+C` quando ver "Application startup complete".

**Verificar estado dos containers:**

```bash
docker compose ps
```

Status esperado: `litellm-postgres (healthy)` e `litellm-gateway (Up, healthy)`.

**Teste local (dentro do WSL ou no PowerShell — graças ao mirrored networking, ambos funcionam):**

```bash
# Dentro do WSL
curl http://localhost:4000/health/liveliness
```

```powershell
# No PowerShell do Windows
Invoke-RestMethod http://localhost:4000/health/liveliness
```

Resposta esperada: `{"status":"healthy"}`.

**Teste de chamada completa com a master key:**

```powershell
$headers = @{
  "Authorization" = "Bearer <MASTER_KEY>"
  "Content-Type"  = "application/json"
}
$body = @{
  model    = "claude-sonnet"
  messages = @(@{role="user"; content="Diga 'oi' em português"})
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" `
  -Method Post -Headers $headers -Body $body
```

Se retornar uma resposta JSON com o texto do Claude, o gateway está funcionando.

## 3.11 Firewall do Windows (LAN → porta 4000)

Mesmo com `networkingMode=mirrored`, o firewall do Windows continua filtrando. Crie regra permitindo a porta 4000 **somente** vindo da LAN:

**PowerShell como administrador:**

```powershell
New-NetFirewallRule `
  -DisplayName "LiteLLM Gateway (LAN only)" `
  -Direction Inbound `
  -LocalPort 4000 `
  -Protocol TCP `
  -Action Allow `
  -Profile Private,Domain `
  -RemoteAddress 10.150.0.0/24
```

**Verificação crítica** — interface deve estar classificada como "Privada":

```powershell
Get-NetConnectionProfile
```

Se aparecer "Public", mude:

```powershell
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
```

**Verificar a regra criada:**

```powershell
Get-NetFirewallRule -DisplayName "LiteLLM Gateway (LAN only)" | Format-List
```

**Teste a partir de outro PC na LAN:**

```powershell
# Em qualquer notebook na rede 10.150.0.0/24
Test-NetConnection -ComputerName 10.150.0.69 -Port 4000
```

`TcpTestSucceeded : True` confirma que a regra está ativa e o `networkingMode=mirrored` está expondo a porta do container.

## 3.12 Geração de virtual keys individuais para devs

Cada dev recebe uma chave única, com permissões e budget próprios. Os containers já estão de pé — basta chamar o endpoint admin.

### Criação via curl/PowerShell

```powershell
$masterKey = "<MASTER_KEY>"

$headers = @{
  "Authorization" = "Bearer $masterKey"
  "Content-Type"  = "application/json"
}

$body = @{
  models = @("claude-sonnet", "gemini-3-1-pro", "gemini-2-5-flash", "qwen-14b-local")
  max_budget = 50.0
  budget_duration = "30d"
  user_id = "joao.silva"
  key_alias = "joao-dev-laptop"
  metadata = @{
    project = "nt-usina"
    role = "developer"
  }
} | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod -Uri "http://localhost:4000/key/generate" `
  -Method Post -Headers $headers -Body $body

Write-Host "Virtual Key: $($response.key)"
Write-Host "Key ID: $($response.key_name)"
```

Saída exemplo:

```
Virtual Key: sk-9KvP3xZ_qN7vR4tY2mFw8GhJ...
Key ID: sk-9KvP3xZ...
```

**Entregue ao dev `joao.silva`:**

- A chave virtual `sk-9KvP3xZ_qN7vR4tY2mFw8GhJ...`
- O endpoint: `http://10.150.0.69:4000`
- A lista de modelos disponíveis para ele

### Perfis sugeridos por papel

**Dev Junior** (acesso restrito a modelos baratos + locais, budget baixo):

```powershell
$body = @{
  models = @("claude-sonnet", "gemini-2-5-flash", "qwen-7b-local", "qwen-14b-local")
  max_budget = 25.0
  budget_duration = "30d"
  user_id = "junior.dev"
  key_alias = "junior-laptop"
} | ConvertTo-Json -Depth 5
```

**Dev Senior** (acesso completo, budget maior):

```powershell
$body = @{
  models = @("claude-opus", "claude-sonnet", "gemini-3-1-pro", "gemini-2-5-pro", "gemini-2-5-flash", "deepseek-local", "qwen-14b-local", "qwen-7b-local")
  max_budget = 150.0
  budget_duration = "30d"
  user_id = "senior.dev"
  key_alias = "senior-laptop"
} | ConvertTo-Json -Depth 5
```

**Privacy/SERIN** (apenas modelos locais, sem budget pois é gratuito):

```powershell
$body = @{
  models = @("deepseek-local", "qwen-14b-local", "qwen-7b-local")
  user_id = "serin.dev"
  key_alias = "serin-projeto-confidencial"
} | ConvertTo-Json -Depth 5
```

### Operações em chaves existentes

**Listar todas as chaves:**

```powershell
Invoke-RestMethod -Uri "http://localhost:4000/key/info" -Headers $headers
```

**Atualizar uma chave (ex: aumentar budget):**

```powershell
$updateBody = @{
  key = "sk-9KvP3xZ..."
  max_budget = 100.0
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:4000/key/update" `
  -Method Post -Headers $headers -Body $updateBody
```

**Revogar/deletar uma chave (ex: dev saiu da empresa):**

```powershell
$deleteBody = @{
  keys = @("sk-9KvP3xZ...")
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:4000/key/delete" `
  -Method Post -Headers $headers -Body $deleteBody
```

## 3.13 Acesso e uso do dashboard do LiteLLM

### Acesso

A partir de qualquer máquina na rede local 10.150.0.0/24:

```
http://10.150.0.69:4000/ui
```

A partir do próprio servidor:

```
http://localhost:4000/ui
```

### Login

- **Username:** `admin`
- **Password:** sua **master key** (`<MASTER_KEY>` definida em 3.8)

### Recursos disponíveis

**Tab "Virtual Keys":** listar, criar, editar, revogar, ver gasto acumulado

**Tab "Usage":** gráficos de uso por modelo/dia/semana/mês, custo total em USD, distribuição por `user_id`

**Tab "Models":** lista de todos os modelos configurados, health check, latência média, taxa de erro

**Tab "Logs":** histórico de requisições (input, output, modelo, custo, latência) — **atenção LGPD** se houver dados sensíveis

**Tab "Settings":** adicionar/remover modelos sem editar `config.docker.yaml` (gravado no Postgres), webhooks de alerta de budget

### Recomendações de uso

- **Revisar gastos semanalmente** na aba Usage
- **Configurar alertas de budget** quando uma chave atinge 80% do teto
- **Auditar logs mensalmente**
- **Backup periódico do banco** (seção 6)

---

# Parte 2 — Configuração do Notebook do Dev

Esta seção é o que você entrega para cada desenvolvedor. O único pré-requisito é Node.js — não muda nada em relação à versão NSSM.

> 💡 **Como funciona a integração:** O Claude Code tem suporte oficial a LLM gateways. Basta definir `ANTHROPIC_BASE_URL` apontando para o LiteLLM e `ANTHROPIC_AUTH_TOKEN` com a virtual key do dev. O Claude Code continua funcionando exatamente como sempre — incluindo todos os comandos `/opsx:*` do OpenSpec — sem saber que há um gateway no meio.

## 4.1 Instalação do Node.js

O Claude Code CLI e o OpenSpec CLI exigem Node.js 20.19.0 ou superior.

1. Acesse https://nodejs.org/ e baixe a versão **LTS** mais recente
2. Execute o instalador marcando **"Add to PATH"** e **"Install necessary tools"**
3. Verifique em um **novo** PowerShell:
   ```powershell
   node --version   # deve ser v20.x ou superior
   npm --version
   ```

## 4.2 Instalação do Claude Code CLI

```powershell
npm install -g @anthropic-ai/claude-code
claude --version
```

> ⚠️ **Não configure `ANTHROPIC_API_KEY` no notebook.** O dev nunca precisa da API key real da Anthropic — ele usa exclusivamente a virtual key do LiteLLM via `ANTHROPIC_AUTH_TOKEN`. Se a variável `ANTHROPIC_API_KEY` existir no ambiente do dev, o Claude Code tentará ir direto para a Anthropic, ignorando o gateway.

## 4.3 Configuração das variáveis de ambiente do gateway

**Via PowerShell — nível de usuário:**

```powershell
[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_BASE_URL",
  "http://10.150.0.69:4000",
  "User"
)

[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_AUTH_TOKEN",
  "sk-suaVirtualKeyIndividual...",
  "User"
)
```

**Verificação** (em um novo PowerShell):

```powershell
[Environment]::GetEnvironmentVariable("ANTHROPIC_BASE_URL", "User")
[Environment]::GetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "User")
```

## 4.4 Configuração persistente no perfil do PowerShell

```powershell
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -Force }
notepad $PROFILE
```

Adicione ao final:

```powershell
# ===== LiteLLM Gateway =====
$env:ANTHROPIC_BASE_URL  = "http://10.150.0.69:4000"
$env:ANTHROPIC_AUTH_TOKEN = "sk-suaVirtualKeyIndividual..."
$env:ANTHROPIC_MODEL = "claude-sonnet"

# Aliases de conveniência
function Use-ClaudeOpus    { $env:ANTHROPIC_MODEL = "claude-opus";      Write-Host "Modelo: claude-opus" }
function Use-ClaudeSonnet  { $env:ANTHROPIC_MODEL = "claude-sonnet";    Write-Host "Modelo: claude-sonnet" }
function Use-Gemini25Pro   { $env:ANTHROPIC_MODEL = "gemini-2-5-pro";   Write-Host "Modelo: gemini-2-5-pro" }
function Use-Gemini25Flash { $env:ANTHROPIC_MODEL = "gemini-2-5-flash"; Write-Host "Modelo: gemini-2-5-flash" }
function Use-Qwen14B       { $env:ANTHROPIC_MODEL = "qwen-14b-local";   Write-Host "Modelo: qwen-14b-local (local)" }
function Use-QwenSerin     { $env:ANTHROPIC_MODEL = "deepseek-local";   Write-Host "Modelo: deepseek-local (local/privado)" }
# ===========================
```

Recarregue:

```powershell
. $PROFILE
```

## 4.5 Instalação e configuração do OpenSpec

### Instalação global do CLI

```powershell
npm install -g @fission-ai/openspec
openspec --version
```

### Inicialização em cada projeto

```powershell
cd C:\projetos\meu-projeto
openspec init
```

- **Tools:** `claude`
- **Profile:** `core`
- **Delivery:** `both`

### Configuração do `openspec/config.yaml`

```yaml
schema: spec-driven

context: |
  Stack: Laravel, PostgreSQL, Vue.js, GCP
  Linguagem de código: português brasileiro
  Padrão de branches: feature/*, develop, main
  Merge: --no-ff obrigatório
  Testes: PHPUnit (Laravel), Vitest (Vue)

rules:
  proposal:
    - Incluir impacto em módulos existentes
    - Identificar tabelas do PostgreSQL afetadas
  specs:
    - Usar formato Dado/Quando/Então para cenários
  design:
    - Incluir diagrama de sequência para fluxos com 3+ etapas
  tasks:
    - Tarefas atômicas (máximo 2h cada)
    - Incluir critério de aceite por tarefa
```

### Ativar perfil expandido (opcional)

```powershell
openspec config profile   # selecione "workflows"
openspec update
```

## 4.6 Teste de conexão

### Teste direto do gateway

```powershell
$headers = @{
  "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN"
  "Content-Type"  = "application/json"
}
$body = @{
  model    = "claude-sonnet"
  messages = @(@{role="user"; content="Responda apenas 'Gateway OK' em português"})
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "http://10.150.0.69:4000/v1/chat/completions" `
  -Method Post -Headers $headers -Body $body
```

### Teste com Claude Code

```powershell
claude --print "Diga apenas 'Claude Code via LiteLLM funcionando' em português"
```

A resposta deve vir do Claude via gateway. Confirme nos logs do container:

```bash
# No PC servidor, dentro do WSL
docker compose logs -f litellm
```

### Verificar modelos disponíveis

```powershell
Invoke-RestMethod -Uri "http://10.150.0.69:4000/v1/models" `
  -Headers @{ "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN" }
```

## 4.7 Uso prático no dia a dia

### Iniciar uma sessão

```powershell
cd C:\projetos\meu-projeto
claude
```

### Workflow OpenSpec

```
/opsx:explore adicionar autenticação por CPF no módulo de cadastro
/opsx:propose autenticacao-cpf
# revisar artifacts em openspec/changes/autenticacao-cpf/
/opsx:apply
/opsx:sync
/opsx:archive
```

### Trocar de modelo

```powershell
Use-ClaudeOpus
Use-Gemini25Flash
Use-Qwen14B
Use-QwenSerin
claude
```

Ou inline:

```powershell
claude --model gemini-2-5-pro
claude --model qwen-14b-local
claude --model deepseek-local
```

### Verificar gasto da virtual key

```powershell
$headers = @{ "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN" }
$info = Invoke-RestMethod -Uri "http://10.150.0.69:4000/key/info" -Headers $headers
Write-Host "Gasto: `$$($info.info.spend) / `$$($info.info.max_budget)"
```

### Projeto SERIN / dados sensíveis

```powershell
$env:ANTHROPIC_MODEL = "deepseek-local"
claude
```

O LiteLLM (container) roteia para o LM Studio (Windows host) via `host.docker.internal:1234` — os prompts e respostas nunca saem do PC servidor.

---

## 5. Troubleshooting

### Container `litellm-gateway` não fica `healthy`

```bash
docker compose ps
docker compose logs litellm --tail 100
```

**Causas comuns:**

| Sintoma no log | Causa | Solução |
|---|---|---|
| `connection refused` em `postgres:5432` | Container do Postgres ainda não pronto | Aguardar — `depends_on` com `condition: service_healthy` resolve em até 30s |
| `password authentication failed for user "litellm_user"` | `.env` diferente do volume já populado | Apagar volume e recriar: `docker compose down -v && docker compose up -d` |
| `Authentication Error... ANTHROPIC_API_KEY` vazia | `.env` não carregado | Confirmar `docker compose config` e reiniciar |
| `Could not resolve host: host.docker.internal` | `extra_hosts` ausente do compose ou Docker antigo | Atualizar Docker Engine; conferir bloco `extra_hosts` no `docker-compose.yml` |

### Dev não consegue conectar (`Connection refused` / `Timeout`)

1. Containers de pé?
   ```bash
   docker compose ps
   ```
2. Gateway responde no localhost do servidor?
   ```powershell
   Invoke-RestMethod http://localhost:4000/health/liveliness
   ```
3. Mirrored networking ativo?
   ```powershell
   Get-Content "$env:USERPROFILE\.wslconfig"
   # deve conter: networkingMode=mirrored
   wsl --shutdown   # após qualquer alteração
   ```
4. Firewall do Windows permitindo a conexão?
   ```powershell
   Get-NetFirewallRule -DisplayName "LiteLLM Gateway (LAN only)" | Format-List
   ```
5. Dev alcança a porta?
   ```powershell
   Test-NetConnection -ComputerName 10.150.0.69 -Port 4000
   ```

### Claude Code não usa o gateway

- `ANTHROPIC_BASE_URL` definida no nível de usuário?
  ```powershell
  echo $env:ANTHROPIC_BASE_URL
  ```
- `ANTHROPIC_API_KEY` definida no ambiente? — Claude Code prefere ela. Remova:
  ```powershell
  [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "User")
  [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "Machine")
  ```
- Recarregue o perfil: `. $PROFILE`

### Erro 401 Unauthorized

- `ANTHROPIC_AUTH_TOKEN` com virtual key errada → confirme com `echo $env:ANTHROPIC_AUTH_TOKEN`
- Virtual key revogada → solicitar nova ao administrador (3.12)
- Modelo fora da lista de permissões da chave → administrador deve atualizar permissões via `/key/update`

### Erro 429 Rate Limit

Vem do provider (Anthropic/Gemini), não do LiteLLM. Soluções:

- Aguardar (rate limit típico expira em 1 minuto)
- O LiteLLM aciona o fallback automático definido no `config.docker.yaml`
- Trocar manualmente: `claude --model gemini-2-5-pro`

### Modelo local não responde (`qwen-*-local`, `deepseek-local`)

1. LM Studio rodando? Verifique o ícone na bandeja do Windows
2. Server do LM Studio ativo?
   ```powershell
   # No Windows
   Invoke-RestMethod http://localhost:1234/v1/models
   ```
3. Container alcança o host?
   ```bash
   docker compose exec litellm wget -qO- http://host.docker.internal:1234/v1/models
   ```
   Se falhar, revisar `.wslconfig` (`hostAddressLoopback=true`) e o bloco `extra_hosts` no compose.
4. Nome do modelo no `config.docker.yaml` bate com o do LM Studio (aba **My Models**)?

### Containers não sobem após reboot do Windows

Você optou por inicialização manual. Para subir após reboot:

```powershell
# Inicia o WSL (necessário porque ele só sobe sob demanda)
wsl
# Dentro do WSL:
cd /mnt/c/litellm
docker compose up -d
exit
```

Ou em um único comando do PowerShell:

```powershell
wsl -d Ubuntu -- bash -lc "cd /mnt/c/litellm && docker compose up -d"
```

### `docker pull` falha por SSL em rede corporativa

Adicione o CA do proxy ao trust store do Ubuntu (ver final de 3.3). Para `docker push`/`pull` específicos com TLS auto-assinado, configure também `daemon.json`:

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": []
}
EOF
sudo systemctl restart docker
```

### Dashboard não abre / banco recém criado

- URL correta? `http://10.150.0.69:4000/ui`
- `DATABASE_URL` montada no container?
  ```bash
  docker compose exec litellm printenv DATABASE_URL
  ```
- Container migrou o banco?
  ```bash
  docker compose logs litellm | grep -i "prisma\|migration"
  ```

### Comandos `/opsx:*` não aparecem no Claude Code

1. OpenSpec inicializado no projeto?
   ```powershell
   ls .claude\commands\opsx\
   ```
2. Se não existe, dentro do projeto:
   ```powershell
   openspec init
   ```
3. Após mudança de perfil:
   ```powershell
   openspec update
   ```
4. Reinicie o Claude Code.

---

## 6. Comandos de manutenção

### Atualizar o LiteLLM (pull de nova imagem)

```bash
cd /mnt/c/litellm
docker compose pull litellm
docker compose up -d litellm
docker compose logs -f litellm
```

Sem mexer no Postgres e sem perda de chaves/histórico — a imagem é substituída, o volume permanece.

### Backup do banco

```bash
cd /mnt/c/litellm
mkdir -p backups
DATE=$(date +%Y%m%d-%H%M)
docker compose exec -T postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
  > "backups/litellm-${DATE}.sql"
```

Restaurar um backup:

```bash
docker compose exec -T postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" \
  < backups/litellm-20260514-1200.sql
```

> 💡 Automatize via cron dentro do WSL ou via Tarefa Agendada do Windows chamando `wsl -d Ubuntu -- bash -lc "..."`.

### Rotacionar a master key

1. Gerar nova (ver 3.8)
2. Atualizar `LITELLM_MASTER_KEY` no `.env`
3. Recriar o container:
   ```bash
   docker compose up -d --force-recreate litellm
   ```
4. Atualizar gerenciador de senhas

> ⚠️ Trocar a master key **NÃO invalida** as virtual keys já distribuídas. Apenas o acesso administrativo (criar/listar/deletar chaves, dashboard) requer a nova master key.

### Verificar saúde geral

```bash
# Status dos containers
docker compose ps

# Endpoints saudáveis
curl http://localhost:4000/health/liveliness
curl http://localhost:4000/health/readiness

# Saúde de todos os modelos (precisa master key)
curl -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  http://localhost:4000/health
```

### Logs em tempo real

```bash
docker compose logs -f litellm
docker compose logs -f postgres
```

`Ctrl+C` para parar. Para limitar quantidade:

```bash
docker compose logs --tail=200 litellm
```

### Adicionar/remover modelos sem reiniciar

Use o dashboard web (`/ui`) na aba **Models** — alterações são gravadas no Postgres e aplicadas dinamicamente.

Para mudanças permanentes via `config.docker.yaml`, edite o arquivo e recarregue:

```bash
docker compose restart litellm
```

### Parar e remover

```bash
# Parar containers (mantém volumes/dados)
docker compose stop

# Remover containers (mantém volumes/dados)
docker compose down

# Remover TUDO incluindo o volume do Postgres (perde chaves e histórico!)
docker compose down -v
```

### Acessar o Postgres do container

```bash
docker compose exec postgres psql -U litellm_user -d litellm
```

Dentro do `psql`:

```sql
\dt                      -- listar tabelas
SELECT * FROM "LiteLLM_VerificationToken" LIMIT 5;  -- ver virtual keys
\q
```

---

## 7. Checklist final de segurança

Antes de considerar a configuração concluída, confirme:

- [ ] WSL2 instalado com kernel atualizado (`wsl --update`)
- [ ] `.wslconfig` com `networkingMode=mirrored` e `hostAddressLoopback=true`
- [ ] `systemd=true` em `/etc/wsl.conf` dentro do Ubuntu
- [ ] Docker Engine instalado **dentro do WSL2** (não Docker Desktop)
- [ ] `.env` preenchido com API keys reais, master key e senha do Postgres
- [ ] `.env` **não** commitado (verifique `.gitignore`)
- [ ] Master key gerada com mínimo 48 bytes de entropia (CSPRNG)
- [ ] Master key armazenada em gerenciador de senhas
- [ ] `docker compose ps` mostra ambos os containers como `(healthy)`
- [ ] Postgres do gateway **não** publica porta na LAN (apenas `expose: 5432`)
- [ ] Firewall do Windows com regra `LiteLLM Gateway (LAN only)` restrita ao range `10.150.0.0/24`
- [ ] Interface de rede do servidor classificada como **Private**
- [ ] Roteador da rede **não** tem port forwarding na porta 4000
- [ ] LM Studio em `127.0.0.1:1234` (NÃO exposto à LAN) — acessado pelo container via `host.docker.internal`
- [ ] Cada dev tem virtual key **única**, nunca compartilhada
- [ ] Notebooks dos devs têm `ANTHROPIC_BASE_URL` e `ANTHROPIC_AUTH_TOKEN` no nível **usuário**
- [ ] Notebooks dos devs **não** têm `ANTHROPIC_API_KEY` definida
- [ ] Dashboard testado e acessível em `http://10.150.0.69:4000/ui`
- [ ] Claude Code testado em notebook do dev com `claude --print "teste"` passando pelo gateway
- [ ] Rotina de backup `pg_dump` definida (cron do WSL ou Tarefa Agendada do Windows)
- [ ] OpenSpec inicializado em pelo menos um projeto (`openspec init`)
- [ ] Pelo menos 1 dev executou o workflow completo: `/opsx:propose` → `/opsx:apply` → `/opsx:archive`

---

**Documento gerado para Natural Tecnologia / SERIN CGOTIC**
**Stack:** LiteLLM Gateway (Docker/WSL2) + PostgreSQL (container) + LM Studio (Windows host) + Claude Code CLI + OpenSpec (OPSX)
**Versão:** Docker (substitui a versão NSSM nativa documentada em `README.md`)
**Última revisão:** maio 2026
