# n8n with External Task Runners

This repository contains a customized **n8n** deployment optimized for security and scale using **External Task Runners**. It allows for isolated execution of JavaScript and Python code blocks.

---

## 🚀 Key Features

- **Isolated Execution**: Code blocks run in a separate container, shielding the main n8n process.
- **Customized Python Environment**: Pre-loaded with `pandas`, `numpy`, and `yt-dlp`.
- **Media Support**: `ffmpeg` is included in the main container for video/audio processing.
- **Dockerized**: Easy to deploy and manage with Docker Compose.

---

## 🏗️ Architecture Overview

Unlike a standard n8n deployment where code in the "Code Node" runs inside the main application process, this setup uses **External Task Runners**.

### Component Diagram

```mermaid
graph TD
    User([User Browser]) -->|Port 5678| n8n[n8n Main Container]
    n8n -->|Port 5679| Broker[Task Broker]
    Broker --> Runner[Task Runner Container]
    Runner -->|Executes| Code[JS/Python Code]
```

1.  **n8n Main Container**: Handles the UI, workflow logic, and node execution. It exposes the dashboard on port `5678`.
2.  **Task Broker**: An internal service (on port `5679`) that acts as a middleman. When a workflow needs to run code, it sends the task here instead of executing it locally.
3.  **Task Runner Container**: A separate container that polls the Broker for tasks. It executes the JavaScript or Python code in an isolated environment and returns the results.
4.  **Code Execution**: The actual execution happens in the Runner, ensuring that heavy computations or crashes in your code don't affect the stability of the main n8n application.

---

## 📁 Project Structure

- **[docker-compose.yml](./docker-compose.yml)**: Orchestrates n8n and the task-runner services.
- **[Dockerfile.n8n](./Dockerfile.n8n)**: Custom image for the main n8n application (includes `ffmpeg` and local Python libs).
- **[Dockerfile.runners](./Dockerfile.runners)**: Custom image for the external task runners (includes `uv` and specific data science libs).
- **[n8n-task-runners.json](./n8n-task-runners.json)**: Configuration for JS and 
Python runner environments, specifying allowed libraries and environment variables.

---

## 🛠️ Setup & Installation

### 1. Prerequisites
Create a `.env` file in the root directory.

```bash
touch .env
```

Then, edit `.env` and define your variables:
- `N8N_RUNNERS_AUTH_TOKEN`: A secure random secret shared between the app and runner.
- `WEBHOOK_URL`: Your public-facing URL (e.g., from ngrok).
- `N8N_HOST`: Your public domain or localhost.

### 2. Build the Images
Build both the application and runner containers:

```bash
docker compose build
```

### 3. Start the Containers
Launch the services in detached mode:

```bash
docker compose up -d
```

---

## 🌐 External Access (ngrok)

To receive webhooks (like from GitHub, Stripe, or Telegram), n8n must be accessible from the internet. Here is how to set up ngrok:

### 1. Install ngrok
On Debian/Ubuntu based systems:

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
  && echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" \
  | sudo tee /etc/apt/sources.list.d/ngrok.list \
  && sudo apt update \
  && sudo apt install ngrok
```

### 2. Configure Auth Token
Add your ngrok authtoken (get it from the [ngrok dashboard](https://dashboard.ngrok.com/)):

```bash
ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>
```

### 3. Start ngrok
Expose port 5678 to the internet:

```bash
ngrok http 5678
```

### 4. Update n8n Configuration
1.  Copy the forwarding URL shown in the ngrok terminal (e.g., `https://abc-123.ngrok-free.app`).
2.  Update your `WEBHOOK_URL` and `N8N_HOST` in `.env`:
    ```env
    WEBHOOK_URL=https://abc-123.ngrok-free.app/
    N8N_HOST=abc-123.ngrok-free.app
    ```
3.  Restart the containers:
    ```bash
    docker compose up -d
    ```

---

## ⚙️ Configuration & Customization

### Python Dependencies
To add more Python libraries:
1.  Modify `Dockerfile.runners` to include the library in the `uv pip install` command.
2.  Update the `N8N_RUNNERS_EXTERNAL_ALLOW` list in `n8n-task-runners.json` to permit its import.

### Security (Environment Access)
The setup is configured with `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`. Only variables explicitly listed in `n8n-task-runners.json` under `allowed-env` are accessible within code nodes.

---

## 🔍 Troubleshooting

### Missing Auth Token
If you see `Error: Missing auth token` in the logs, ensure `N8N_RUNNERS_AUTH_TOKEN` is set in your `.env` file AND that you have restarted the containers.

### Viewing Logs
- **Main App**: `docker logs -f n8n-main`
- **Runners**: `docker logs -f runner`
