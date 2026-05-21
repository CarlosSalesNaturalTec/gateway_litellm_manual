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
│        │            ▲ 10.150.0.69:1234 (LAN, fw block)   │              │
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

> ⚠️ **Pré-requisito:** a distro precisa já ter sido aberta uma vez para você criar o usuário UNIX inicial (`Enter new UNIX username` / `New password`). Sem isso o `sudo` falha. Se ainda não fez, rode `wsl` no PowerShell, complete o setup do usuário e `exit`.

> ⚠️ **Não use stdin via pipe do PowerShell para o `sudo` do WSL.** Comandos como `@'...'@ | wsl -- sudo tee ...` quebram o prompt interativo da senha (o `sudo` lê a senha de stdin junto com o conteúdo do arquivo e falha). O caminho que funciona é entrar no WSL primeiro e rodar o heredoc bash interativamente, onde o `sudo` consegue pedir a senha no TTY.

**Passo 1 — Entrar no WSL (no PowerShell):**

```powershell
wsl
```

**Passo 2 — Já dentro do Ubuntu, criar o arquivo com heredoc bash:**

```bash
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true

[network]
generateResolvConf=true
EOF
```

O `sudo` vai pedir a senha do seu usuário do Ubuntu no terminal interativo. Depois o `tee` grava em `/etc/wsl.conf` e ecoa o conteúdo no console — isso é esperado.

**Passo 3 — Conferir e sair:**

```bash
cat /etc/wsl.conf
exit
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

### Rede corporativa com SSL inspection (FortiGate, ZScaler, McAfee Web Gateway, etc.)

Aplicável se o `docker pull` falhar com erro do tipo:

```
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

O sintoma é: o firewall corporativo está fazendo **TLS inspection** (man-in-the-middle "do bem"), o Windows confia no CA dele (foi instalado por política de domínio), mas o WSL Ubuntu não — e o Docker usa o trust store do Ubuntu, não do Windows.

**Passo 1 — Descobrir qual CA está interceptando** (PowerShell no Windows):

```powershell
$tcp = New-Object Net.Sockets.TcpClient
$tcp.Connect('ghcr.io', 443)
$ssl = New-Object Net.Security.SslStream($tcp.GetStream(), $false, {$true})
$ssl.AuthenticateAsClient('ghcr.io')
$chain = New-Object Security.Cryptography.X509Certificates.X509Chain
$chain.Build([Security.Cryptography.X509Certificates.X509Certificate2]$ssl.RemoteCertificate) | Out-Null
$chain.ChainElements | ForEach-Object {
  Write-Host ("Subject: " + $_.Certificate.Subject)
  Write-Host ("Issuer : " + $_.Certificate.Issuer)
  Write-Host ("Thumbprint: " + $_.Certificate.Thumbprint)
  Write-Host ""
}
$ssl.Dispose(); $tcp.Close()
```

Numa rede sem inspection o leaf `*.ghcr.io` é emitido por uma CA pública (Let's Encrypt, DigiCert etc.). Com inspection, o issuer revela o produto — ex: `CN=FG100FTK22013189, O=Fortinet` (FortiGate). Anote o **Thumbprint** do(s) CA(s) auto-assinado(s) na cadeia.

**Passo 2 — Exportar o(s) CA(s) do certificate store do Windows para o projeto:**

```powershell
$out = "C:\codeDiversos\gateway_litellm_manual\certs"
New-Item -ItemType Directory -Force -Path $out | Out-Null

# Liste todos os Fortinet/ZScaler/McAfee instalados (ajuste o filtro)
Get-ChildItem Cert:\LocalMachine\Root | Where-Object {
  $_.Subject -like "*Fortinet*" -or $_.Subject -like "*ZScaler*" -or $_.Subject -like "*McAfee*"
} | ForEach-Object {
  $name = ($_.Subject -replace '.*CN=([^,]+).*','$1') -replace '[^A-Za-z0-9]','_'
  $pem = "-----BEGIN CERTIFICATE-----`n" +
         [Convert]::ToBase64String($_.RawData, 'InsertLineBreaks') +
         "`n-----END CERTIFICATE-----`n"
  $path = Join-Path $out "$name.crt"
  [IO.File]::WriteAllText($path, $pem, [Text.UTF8Encoding]::new($false))
  Write-Host "Exportado: $path"
}
```

Resultado: um `.crt` por CA em `C:\codeDiversos\gateway_litellm_manual\certs\`. **Adicione `certs/` ao `.gitignore`** se ainda não estiver — embora os CAs do firewall não sejam segredo, eles identificam o equipamento de rede e não precisam viver no repo.

**Passo 3 — Instalar os CAs no trust store do WSL Ubuntu** (no terminal WSL, interativo):

```bash
sudo cp /mnt/c/codeDiversos/gateway_litellm_manual/certs/*.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates    # esperado: "N added, 0 removed"
sudo systemctl restart docker
```

> ⚠️ **Não pipeie o `sudo` via stdin do PowerShell** (`@'...'@ | wsl -- sudo ...`) — o prompt de senha quebra. Entre no WSL com `wsl` e rode os comandos no terminal interativo.

**Passo 4 — Validar:**

```bash
docker run --rm hello-world           # pull pequeno, falha rápido se TLS ainda quebrado
docker compose pull                   # baixa as imagens do compose
```

Se `hello-world` passar, prossiga com a etapa 3.10 (`docker compose up -d`).

## 3.4 Instalação e configuração do LM Studio (no Windows host)

O LM Studio continua sendo instalado nativamente no Windows porque é GUI e gerencia GPU/CPU nativamente. Os containers do WSL2 acessam o LM Studio através do **IP fixo do PC servidor na LAN (`10.150.0.69:1234`)** — não via `host.docker.internal`.

> ⚠️ **Por que não `host.docker.internal`?** Mesmo com `hostAddressLoopback=true` no `.wslconfig` e `extra_hosts: host.docker.internal:host-gateway` no compose, o nome resolve para o gateway da bridge do Docker (`172.17.0.1`), que é uma interface dentro da VM do WSL — não o loopback do Windows. O `hostAddressLoopback` ajuda a shell do WSL a alcançar o `127.0.0.1` do Windows, mas não estende esse benefício aos containers do Docker (uma camada de rede a mais). A solução prática é deixar o LM Studio escutando em `0.0.0.0:1234` e proteger a porta com firewall (passo abaixo).

1. Acesse https://lmstudio.ai/ e baixe a versão mais recente para Windows
2. Execute o instalador como administrador
3. Aceite as configurações padrão durante a instalação
4. Após a primeira execução, vá em **Settings → General**:
   - Marque **"Run LM Studio at startup"**
   - Marque **"Start minimized to system tray"**
5. Vá em **Settings → Developer**:
   - Marque **"Auto-start LLM server on app launch"**
   - **"Server port"** = `1234`
   - **"Serve on local network"** = **ON** (necessário para o container Docker alcançar o LM Studio via `10.150.0.69:1234`)

### Regra de firewall para a porta 1234

Como o LM Studio agora escuta em `0.0.0.0:1234`, é preciso bloquear o acesso externo na porta 1234 e permitir apenas conexões locais (do próprio Windows e dos containers do Docker via WSL).

**PowerShell como administrador:**

```powershell
# Bloquear porta 1234 vinda da LAN externa
New-NetFirewallRule `
  -DisplayName "LM Studio (block LAN)" `
  -Direction Inbound `
  -LocalPort 1234 `
  -Protocol TCP `
  -Action Block `
  -Profile Private,Domain `
  -RemoteAddress 10.150.0.0/24
```

> 📝 Conexões originadas do próprio host (loopback) e dos containers do Docker rodando dentro do WSL com `networkingMode=mirrored` aparecem para o Windows como tráfego local (origem `10.150.0.69` → destino `10.150.0.69`) — não filtrado por esta regra. Conexões vindas de outros IPs da LAN (`10.150.0.x` ≠ `.69`) ficam bloqueadas.

**Validar a regra:**

```powershell
Get-NetFirewallRule -DisplayName "LM Studio (block LAN)" | Format-List
```

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
4. Anote o **identificador exato** que aparece no LM Studio após o download — necessário no `config.yaml`

> 💡 Para descobrir o identificador exato, vá na aba **My Models** do LM Studio. O nome listado ali é o que o servidor expõe na API.

## 3.6 Servidor do LM Studio acessível ao WSL2

1. No LM Studio, vá na aba **Developer** (ícone `</>`)
2. Clique em **Start Server** (deve ficar verde, indicando "Running on port 1234")
3. Em **Server Options**:
   - **Server Port:** `1234`
   - **JIT (Just-In-Time) Loading:** habilitado
   - **Auto-Unload:** habilitado (descarrega após 10 minutos de inatividade)
   - **CORS:** desabilitado

**Verificar acesso a partir do WSL e a partir de um container** (com o LM Studio rodando):

```bash
# 1) Teste do WSL host — confirma mirrored networking + LM Studio escutando
curl http://localhost:1234/v1/models

# 2) Teste do container — confirma o caminho que o LiteLLM vai usar em produção
docker run --rm curlimages/curl -s http://10.150.0.69:1234/v1/models
```

Ambos devem retornar o JSON com a lista de modelos baixados.

> ⚠️ **Não use `host.docker.internal` como teste** — ele resolve para o gateway da bridge do Docker (`172.17.0.1`), que **não é** o loopback do Windows e vai dar `Connection refused`. O caminho correto é via o IP fixo do PC servidor na LAN (`10.150.0.69`).

**Se o teste 1 falhar:** confirme em `.wslconfig` que `networkingMode=mirrored` e `[experimental] hostAddressLoopback=true` estão presentes, e reinicie o WSL (`wsl --shutdown` + reabrir).

**Se o teste 2 falhar (mas o 1 passar):** confirme que **"Serve on local network = ON"** no LM Studio (Settings → Developer) — sem isso ele só escuta em `127.0.0.1` e o container não alcança.

## 3.7 Preparar a pasta do projeto e os arquivos do compose

> 📌 **Convenção de caminho neste manual.** O diretório do projeto (com `docker-compose.yml`, `config.yaml`, `.env`) está em:
>
> - **Windows:** `C:\codeDiversos\gateway_litellm_manual` → referenciado nas instruções PowerShell como `$PROJECT_DIR` (defina uma vez por sessão: `$PROJECT_DIR = "C:\codeDiversos\gateway_litellm_manual"`)
> - **WSL:** `/mnt/c/codeDiversos/gateway_litellm_manual` (a unidade `C:` é montada automaticamente em `/mnt/c/`)
>
> Se você optar por colocar o projeto em outro lugar (ex: `~/litellm` no home do WSL), ajuste os caminhos das instruções abaixo. **Não use `C:\litellm`** — essa pasta contém a versão antiga (NSSM/Python nativo) e foi substituída por este guia.

Qualquer pasta sob `C:\` é montada automaticamente dentro do WSL em `/mnt/c/...` — assim você edita arquivos com VS Code/Notepad no Windows e o container vê tudo.

> ⚠️ **Performance:** rodar containers Docker com volumes montados em `/mnt/c/...` é mais lento que mover o projeto para o filesystem nativo do WSL (ex: `~/litellm`). Para este caso (config + um YAML), a diferença é irrelevante. Para projetos maiores, mova para o home do WSL.

**No WSL, navegue até o projeto:**

```bash
cd /mnt/c/codeDiversos/gateway_litellm_manual
ls -la
```

Os três arquivos necessários já existem no repositório:

| Arquivo | Função |
|---|---|
| `docker-compose.yml` | Definição dos serviços `postgres` e `litellm` |
| `config.yaml` | Configuração do LiteLLM (modelos, fallbacks, routing) com `api_base` para LM Studio em `http://10.150.0.69:1234/v1` |
| `.env.example` | Template das variáveis (Anthropic, Gemini, master key, credenciais Postgres) |

> 📝 O `docker-compose.yml` deste repositório já monta `./config.yaml` direto — nenhum renomeio necessário. O `api_base` dos modelos locais aponta para `http://10.150.0.69:1234/v1`; se o IP fixo do seu PC servidor for diferente, ajuste no `config.yaml` antes de subir os containers.

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
cd /mnt/c/codeDiversos/gateway_litellm_manual
cp .env.example .env
```

**Abra `.env`** (em Windows: `C:\codeDiversos\gateway_litellm_manual\.env`; em WSL: `nano .env`) e preencha:

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

Ainda no WSL, dentro de `/mnt/c/codeDiversos/gateway_litellm_manual`:

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

Resposta esperada: `I'm alive!` (texto puro). Para checagem mais completa (incluindo conexão com o Postgres), use `/health/readiness` — que retorna um JSON com `db = connected`.

**Teste de chamada completa com a master key:**

```powershell
# Lê a master key direto do .env (evita colar a chave em texto plano no histórico do PowerShell)
$envPath = "C:\codeDiversos\gateway_litellm_manual\.env"
$mk = (Select-String -Path $envPath -Pattern '^LITELLM_MASTER_KEY=').Line.Substring('LITELLM_MASTER_KEY='.Length)

$headers = @{
  "Authorization" = "Bearer $mk"
  "Content-Type"  = "application/json"
}
$body = @{
  model    = "claude-sonnet"
  messages = @(@{role="user"; content="Diga 'oi' em português"})
} | ConvertTo-Json -Depth 5

$resp = Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" `
  -Method Post -Headers $headers -Body $body

# Texto da resposta do modelo
$resp.choices[0].message.content

# (opcional) JSON completo com metadados de uso/roteamento
$resp | ConvertTo-Json -Depth 10
```

> 💡 **Por que extrair `choices[0].message.content`?** O `Invoke-RestMethod` desserializa o JSON num objeto e exibe apenas o primeiro nível em formato tabular — `message=` aparece colapsado. O texto real do modelo vive em `choices[0].message.content`. Sem a extração, parece que a chamada não trouxe resposta.

**Validar os demais providers** trocando o `model` e refazendo a chamada (a mesma `$headers` continua válida):

```powershell
# Gemini (Vertex AI) — confirma a chave GEMINI_API_KEY e o roteamento Vertex
$body = @{ model = "gemini-3-1-pro"; messages = @(@{role="user"; content="Diga 'oi' em português"}) } | ConvertTo-Json -Depth 5
(Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" -Method Post -Headers $headers -Body $body).choices[0].message.content

# Modelo local via LM Studio — confirma o caminho container → 10.150.0.69:1234
$body = @{ model = "qwen-14b-local"; messages = @(@{role="user"; content="Diga 'oi' em português"}) } | ConvertTo-Json -Depth 5
(Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" -Method Post -Headers $headers -Body $body).choices[0].message.content
```

> ⚠️ **Se preferir colar a chave inline** (ex: testando de outra máquina sem acesso ao `.env`), substitua a primeira linha por `$mk = "sk-suaMasterKeyReal..."`. **Nunca** deixe o placeholder literal `<MASTER_KEY>` — o LiteLLM responde 401 com mensagem `LiteLLM Virtual Key expected. Received=<MAS****KEY>, expected to start with 'sk-'`.

Se cada chamada imprimir o texto do modelo correspondente, o gateway está roteando corretamente para os três providers (Anthropic, Gemini, LM Studio).

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
# Mesma técnica de 3.10: lê do .env (ou substitua por $masterKey = "sk-...")
$envPath = "C:\codeDiversos\gateway_litellm_manual\.env"
$masterKey = (Select-String -Path $envPath -Pattern '^LITELLM_MASTER_KEY=').Line.Substring('LITELLM_MASTER_KEY='.Length)

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
Invoke-RestMethod -Uri "http://localhost:4000/key/list" -Headers $headers
```

**Ver detalhes de uma chave específica** (gasto acumulado, modelos permitidos, budget):

```powershell
$key = "sk-9KvP3xZ..."   # substitua pela chave virtual
Invoke-RestMethod -Uri "http://localhost:4000/key/info?key=$key" -Headers $headers
```

> ⚠️ `/key/info` **sem** o parâmetro `?key=...` retorna `404 Key not found in database` — ele só responde sobre uma chave específica, não lista todas. Use `/key/list` para listar.

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

**Tab "Settings":** adicionar/remover modelos sem editar `config.yaml` (gravado no Postgres), webhooks de alerta de budget

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

O LiteLLM (container) roteia para o LM Studio (Windows host) via `10.150.0.69:1234` — os prompts e respostas nunca saem do PC servidor (a porta 1234 fica bloqueada para a LAN externa pelo firewall do Windows configurado em 3.4).

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
| `Connection refused` em `10.150.0.69:1234` | LM Studio com "Serve on local network=OFF" ou parado | Em LM Studio → Developer → Server Options: ligar "Serve on local network", confirmar Start Server verde |

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
- O LiteLLM aciona o fallback automático definido no `config.yaml`
- Trocar manualmente: `claude --model gemini-2-5-pro`

### Modelo local não responde (`qwen-*-local`, `deepseek-local`)

1. LM Studio rodando? Verifique o ícone na bandeja do Windows
2. Server do LM Studio ativo?
   ```powershell
   # No Windows
   Invoke-RestMethod http://localhost:1234/v1/models
   ```
3. Container alcança o LM Studio?
   ```bash
   docker compose exec litellm wget -qO- http://10.150.0.69:1234/v1/models
   ```
   Se falhar, confirme:
   - LM Studio em **Settings → Developer → "Serve on local network = ON"**
   - Server está em "Running on port 1234" na aba Developer
   - A regra de firewall `LM Studio (block LAN)` (seção 3.4) não está bloqueando o próprio host — o tráfego do container via NAT do WSL deve aparecer como originado de `10.150.0.69`, não como remote LAN
4. Nome do modelo no `config.yaml` bate com o que aparece na aba **My Models** do LM Studio?

### Containers não sobem após reboot do Windows

Você optou por inicialização manual. Para subir após reboot:

```powershell
# Inicia o WSL (necessário porque ele só sobe sob demanda)
wsl
# Dentro do WSL:
cd /mnt/c/codeDiversos/gateway_litellm_manual
docker compose up -d
exit
```

Ou em um único comando do PowerShell:

```powershell
wsl -d Ubuntu -- bash -lc "cd /mnt/c/codeDiversos/gateway_litellm_manual && docker compose up -d"
```

### `docker pull` falha por SSL em rede corporativa

Sintoma típico: `tls: failed to verify certificate: x509: certificate signed by unknown authority` ao puxar de `ghcr.io`, `docker.io`, etc. Causa: firewall corporativo (FortiGate/ZScaler/McAfee Web Gateway) fazendo TLS inspection — o Windows confia no CA dele, o WSL não.

**Solução completa:** ver subseção [Rede corporativa com SSL inspection](#rede-corporativa-com-ssl-inspection-fortigate-zscaler-mcafee-web-gateway-etc) ao final de 3.3 (descobrir o CA pela cadeia TLS → exportar do store do Windows → instalar no trust store do WSL → reiniciar Docker).

Para registries com TLS auto-assinado fora da inspection (raro neste setup), configure também `daemon.json`:

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
cd /mnt/c/codeDiversos/gateway_litellm_manual
docker compose pull litellm
docker compose up -d litellm
docker compose logs -f litellm
```

Sem mexer no Postgres e sem perda de chaves/histórico — a imagem é substituída, o volume permanece.

### Backup do banco

```bash
cd /mnt/c/codeDiversos/gateway_litellm_manual
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

Para mudanças permanentes via `config.yaml`, edite o arquivo e recarregue:

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
- [ ] LM Studio em `0.0.0.0:1234` com "Serve on local network = ON" — acesso da LAN **bloqueado** pela regra `LM Studio (block LAN)` no firewall do Windows (apenas o próprio host e os containers do Docker no WSL alcançam)
- [ ] `config.yaml` com `api_base: http://10.150.0.69:1234/v1` para os modelos locais (não `host.docker.internal`)
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
