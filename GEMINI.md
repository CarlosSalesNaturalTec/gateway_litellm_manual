# LiteLLM Gateway (Docker/WSL2) + LM Studio

## Project Overview
This project is an infrastructure configuration repository for running a **LiteLLM Gateway** shared across a local network. It acts as a secure, centralized proxy for interacting with external AI APIs (Anthropic, Google Gemini) and local open-source models (via LM Studio). 

The architecture is explicitly designed for a **Windows 11 / WSL2** environment:
- **LiteLLM Gateway (`litellm-gateway`)**: Runs in a Docker container within WSL2 (Ubuntu), exposed on port 4000.
- **PostgreSQL (`litellm-postgres`)**: Runs in a Docker container alongside LiteLLM to store virtual keys, usage metrics, and configurations.
- **LM Studio**: Runs natively on the Windows host, serving local quantized models (e.g., DeepSeek, Qwen) on port 1234.
- **Client Side**: Developers use `Claude Code CLI` and `OpenSpec` on their local machines, pointing `ANTHROPIC_BASE_URL` to this gateway.

## Main Technologies
- **Docker Compose**: Orchestrates the LiteLLM and PostgreSQL containers.
- **LiteLLM**: The core gateway (`ghcr.io/berriai/litellm-database`).
- **PostgreSQL 16**: Database for LiteLLM.
- **LM Studio**: Local LLM runner.
- **WSL2 (Ubuntu)**: The environment hosting the Docker Engine.

## Building and Running

The project relies on Docker Compose to start the services. 

* **Start the Gateway (in background):**
  ```bash
  docker compose up -d
  ```
* **View Logs:**
  ```bash
  docker compose logs -f
  # or specifically for litellm
  docker compose logs -f litellm
  ```
* **Stop the Gateway:**
  ```bash
  docker compose stop
  ```
* **Tear down containers (keeps volumes):**
  ```bash
  docker compose down
  ```
* **Reset Database (WARNING: Deletes all data, virtual keys, and history!):**
  ```bash
  docker compose down -v
  ```
* **Check Health:**
  ```bash
  curl http://localhost:4000/health/liveliness
  curl http://localhost:4000/health/readiness
  ```

## Configuration & Key Files

- **`docker-compose.yml`**: Defines the `litellm` and `postgres` services. Configured for local internal communication and exposes port `4000` to the WSL/Windows host.
- **`config.yaml`**: The main LiteLLM routing configuration. It defines the models (Claude, Gemini, Local models via LM Studio), routing strategies, and fallbacks. Local models are routed to the Windows host IP (`http://10.150.0.69:1234/v1`).
- **`.env`**: Contains sensitive keys (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `LITELLM_MASTER_KEY`) and database credentials. **Never commit this file.** Use `.env.example` as a template.

## Infrastructure & Development Conventions

- **Mirrored Networking**: The setup strongly relies on WSL2's `networkingMode=mirrored` (configured in `~/.wslconfig`) to expose the container's port 4000 directly to the Windows host and the local LAN (10.150.0.0/24).
- **Windows Firewall**: Firewall rules are critical in this setup to restrict access to port 4000 (LAN only) and port 1234 (blocked from external LAN, accessible only by host/WSL).
- **API Key Security**: Real API keys are only kept in the server's `.env`. Developers are issued individual **Virtual Keys** (`sk-...`) via the LiteLLM admin dashboard or API. Developers configure their environments using `ANTHROPIC_AUTH_TOKEN=<virtual_key>` and `ANTHROPIC_BASE_URL=http://<server-ip>:4000`.
- **Certificates**: If dealing with corporate SSL inspection, custom root certificates may be placed in the `certs/` directory (which should be ignored by git).
