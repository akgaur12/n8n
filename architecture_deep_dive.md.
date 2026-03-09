# n8n Architecture Deep Dive

This project is a customized, high-security deployment of **n8n**, a powerful workflow automation tool. Unlike a standard installation, this setup uses **External Task Runners** to isolate code execution.

---

## 🏗️ The Big Picture

Most n8n setups run everything (including your custom JavaScript/Python code) inside one single container. This project splits it into three specialized parts:

```mermaid
graph TD
    subgraph "External World"
        Web[Webhooks/API] --> Ng[ngrok]
    end

    subgraph "Docker Network"
        Ng -->|Proxy| N8N[n8n-main]
        N8N -->|Tasks| Broker[Internal Broker]
        Broker <--> Runner[Task Runner]
    end

    subgraph "Execution"
        Runner -->|Isolates| Code[JS/Python Code]
    end
```

---

## 🧩 Component Breakdown

### 1. `n8n-main` (The Brain)
- **Role**: Handles the UI, workflow logic, triggers, and data storage.
- **File**: Defined in [Dockerfile.n8n](file:///home/akash/GenAI/n8n/Dockerfile.n8n).
- **Customization**: It includes `ffmpeg` (for media processing) and basic Python libraries. Even though it *can* run code, we've told it to send code tasks to the Runner instead.

### 2. `task-runners` (The Brawn)
- **Role**: Executes the code blocks in your workflows.
- **File**: Defined in [Dockerfile.runners](file:///home/akash/GenAI/n8n/Dockerfile.runners).
- **Isolation**: If a Python script crashes or consumes too much memory, it only affects the Runner container, leaving your main n8n instance perfectly stable.

### 3. `ngrok` (The Bridge)
- **Role**: Creates a secure tunnel from the internet to your local machine.
- **Why?**: This allows external services (like GitHub, Stripe, or Telegram) to send data (webhooks) to your local n8n instance.

---

## 📁 File-by-File Detail

| File | Purpose | Key Detail |
| :--- | :--- | :--- |
| **[docker-compose.yml](file:///home/akash/GenAI/n8n/docker-compose.yml)** | System Orchestrator | Connects all containers, manages ports (5678 for UI, 5679 for Broker), and sets environment variables. |
| **[n8n-task-runners.json](file:///home/akash/GenAI/n8n/n8n-task-runners.json)** | The Rulebook | Defines exactly what libraries (like `numpy`, `pandas`) code blocks are allowed to use. |
| **[Dockerfile.runners](file:///home/akash/GenAI/n8n/Dockerfile.runners)** | Runner Recipe | A slim image using `uv` (a fast Python package manager) to install your data science tools. |
| **[.env](file:///home/akash/GenAI/n8n/.env)** | The Vault | Stores sensitive items like your `NGROK_AUTHTOKEN` and the shared `N8N_RUNNERS_AUTH_TOKEN`. |

---

## 🔄 Workflow: Running a "Code Node"

1.  **Trigger**: You start a workflow that contains a **Code Node** (Python or JS).
2.  **Request**: `n8n-main` sees the code but doesn't run it. It sends the code and the data to the **Internal Broker**.
3.  **Handoff**: the `task-runner` container polls the broker, grabs the task, and runs it in an isolated sandbox.
4.  **Security Check**: The runner checks [n8n-task-runners.json](file:///home/akash/GenAI/n8n/n8n-task-runners.json) to ensure the code isn't trying to access forbidden environment variables or libraries.
5.  **Return**: Once finished, the runner sends the result back to `n8n-main`, which continues the workflow.

---

## 🛡️ Why this setup?

- **Security**: Code nodes can't "escape" to read your server's sensitive files.
- **Scalability**: You could theoretically run multiple Runner containers if you have heavy workloads.
- **Stability**: Bad code won't take down your entire automation server.
- **Convenience**: `ngrok` is built-in, so you don't have to worry about port forwarding or complex networking.
