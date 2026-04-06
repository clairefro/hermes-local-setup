# Hermes Agent: Local Setup

Containerized setup for **Hermes Agent**, the self-improving AI assistant by Nous Research. This configuration is tuned for **local-first privacy** using LM Studio as the LLM provider.

## Quick Start

1. **Start your LLM:**
   - Open **LM Studio** and load `Qwen2.5-Coder-7B-Instruct-GGUF`
   - Start the **Local Server** on port `1234`
   - Ensure CORS is enabled

2. **Launch the Agent**

   ```bash
   docker compose run --rm hermes
   ```

3. **Bootstrap**
   On first run, Hermes launches an interactive setup wizard. Since the environment variables are pre-set in `docker-compose.yml`, you can simply verify the connection and select your model.
4. **Explore Files:**
   Open the `hermes_data/` folder on your host machine. Any code, logs, or persistent memories the agent creates will appear here instantly.

---

## Switching Providers

### Option A: Using Ollama (Local)

If you prefer **Ollama** over LM Studio, update the `environment` section in your `docker-compose.yml`:

```yaml
environment:
  - OPENAI_BASE_URL=http://host.docker.internal:11434/v1
  - OPENAI_API_KEY=ollama # placeholder
```

_Note: Ensure you have run `ollama pull <model>` on your host first._

### Option B: Using Cloud Models (OpenRouter / OpenAI)

To use a cloud provider for more complex reasoning:

1. **Update Compose:**
   ```yaml
   environment:
     - OPENAI_API_KEY=your_sk_key_here
     # Comment out or remove OPENAI_BASE_URL to use the default provider
   ```
2. **In-Agent Command:**
   Inside the running agent terminal, you can type `/model` to switch providers via the TUI without restarting the container.

---

## Data & Privacy

- **Persistent Memory:** All "skills" and "memories" the agent learns are saved to `hermes_data/`.
- **Git Safety:** The `.gitignore` inside `hermes_data/` ensures your private conversations and agent-generated code stay off GitHub while keeping the folder structure intact for new clones.
- **Structure:**
  - `hermes_data/` - The workspace (Mounted Volume).
  - `docker-compose.yml` - Container orchestration.
