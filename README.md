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
   On first run, Hermes launches an interactive setup wizard. Follow the prompts to select your model and verify the connection.
4. **Explore Files:**
   Agent output (code, docs, artifacts) appears in the `output/` folder on your host. Agent config, memories, and skills are in `hermes_data/`.

---

## Getting the Agent to Generate Output Files

Files the agent writes are persisted to your host via the `output/` volume (mapped to `/app/data` inside the container). By default the agent doesn't know this path — you need to tell it once via `SOUL.md`, which is injected into every message as a standing instruction.

### 1. Update SOUL.md

Open `hermes_data/SOUL.md` and add the following (create the file if it doesn't exist):

```markdown
## Environment

When writing files, always use `/app/data/` as the workspace root. This path is mounted to the user's host machine, so anything written here is persisted. Do not write files to any other path unless explicitly instructed.
```

SOUL.md is reloaded every message — no container restart needed.

### 2. Smoke Test

Verify everything is wired up by prompting Hermes:

> Write a file called `hello.txt` with the content "Hello from Hermes!". Confirm the full path you wrote it to.

Then on your host:

```bash
cat output/hello.txt
# Hello from Hermes!
```

### Practical Example

> Research the history of the Linux kernel and write a detailed summary to `/app/data/linux_kernel_history.md`.

The file will appear at `output/linux_kernel_history.md` on your host as soon as the agent writes it.

---

## Re-running the Setup Wizard

The `hermes_data/` directory is gitignored — only the empty directory placeholder is committed. Hermes will run the setup wizard automatically on first launch and generate a fresh `config.yaml` locally.

To reconfigure at any time (change provider, model, API key, etc.):

- **From inside the agent:** Type `/model` in the chat to interactively switch your provider or model without restarting.
- **Full wizard re-run:** Delete the config and relaunch:
  1. ```bash
     rm hermes_data/config.yaml
     ```
  2. ```bash
     docker compose run --rm hermes
     ```

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

- **Output files:** Anything the agent writes (code, docs, artifacts) goes to `output/` on your host, mapped to `/app/data` inside the container.
- **Agent state:** Config, memories, skills, and logs are stored in `hermes_data/`, mapped to `/opt/data` inside the container.
- **Git Safety:** Both `output/` and `hermes_data/` contents are gitignored — none of your conversations, credentials, or agent-generated files will be committed to GitHub. Only the empty directory placeholders are tracked.
- **Structure:**
  - `hermes_data/` - Agent workspace (config, memories, skills). Populated at runtime, never committed.
  - `output/` - Agent-generated files. Populated at runtime, never committed.
  - `docker-compose.yml` - Container orchestration.
