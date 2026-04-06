# Hermes Agent: Local Setup

Containerized setup for **Hermes Agent**, the self-improving AI assistant by Nous Research. This configuration is tuned for **local-first privacy** using LM Studio as the LLM provider.

## Quick Start

1. **Start your LLM:**
   - Open **LM Studio** and load a model (ex: `Qwen2.5-Coder-7B-Instruct-GGUF`) (see [Recommended Local Models](#recommended-local-models))
   - Start the **Local Server** on port `1234`
   - Ensure CORS is enabled
   - Set **Context Length** to at least **17000** tokens (the Hermes system prompt alone is ~17K tokens). Qwen2.5-Coder-7B supports up to 32K — setting it to `32768` gives the most headroom for long conversations.

2. **Launch the Agent**

   ```bash
   docker compose run --rm hermes
   ```

3. **Bootstrap**
   On first run, Hermes launches an interactive setup wizard. Follow the prompts to select your model and verify the connection.
4. **Explore Files:**
   Agent output (code, docs, artifacts) appears in the `working_dir/` folder on your host. Agent config, memories, and skills are in `hermes_data/`.

---

## Getting the Agent to Generate Output Files

Files the agent writes are persisted to your host via the `working_dir/` volume (mapped to `/app/data` inside the container). By default the agent doesn't know this path — you need to tell it once via `hermes_data/SOUL.md`, which is injected into every message as a standing instruction.

`working_dir/` is also a two-way bridge — you can drop any file into it from your host and ask the agent to read or work with it (e.g. _"Summarize the file `report.pdf`"_).

### 1. Update SOUL.md

Open `hermes_data/SOUL.md` and add the following (create the file if it doesn't exist):

```markdown
## Environment

You are running inside a Docker container with two mounted volumes:

- `/app/data/` — your working directory for all artifacts. Read and write files here. This is mounted to the user's host machine and persisted across sessions.
- `/opt/data/` — Hermes configuration and state (config, memories, skills). Managed by Hermes internally — do not write arbitrary files here.

**Default rule:** When producing any user-facing artifact (documents, code, data files, reports, etc.), always write to `/app/data/` unless the user explicitly specifies otherwise. To create a directory structure, write files to their full nested paths (e.g. `/app/data/project/subdir/file.md`) — directories are created automatically.

**Transparency rule:** Every time you write or read a file, explicitly state the full path in your response (e.g. "Writing to `/app/data/report.md`…"). This allows the user to catch and correct wrong paths immediately.
```

SOUL.md is reloaded every message — no container restart needed.

### 2. Smoke Test

Verify everything is wired up by prompting Hermes:

> Write a file called `hello.txt` with the content "Hello from Hermes!". Confirm the full path you wrote it to.

Then on your host:

```bash
cat working_dir/hello.txt
# Hello from Hermes!
```

### Practical Example

> Research the history of the Linux kernel and write a detailed summary named `linux_kernel_history.md`.

The file will appear at `working_dir/linux_kernel_history.md` on your host as soon as the agent writes it.

---

## Hermes Tips

- **Start a new chat thread:** Type `/reset` in the agent chat to clear the conversation context and start fresh. Persistent memories and skills are preserved.
- **Switch model/provider:** Type `/model` to interactively change your model or provider without restarting the container.

---

## Re-running the Setup Wizard

The `hermes_data/` directory is gitignored — only the empty directory placeholder is committed. Hermes will run the setup wizard automatically on first launch and generate a fresh `config.yaml` locally.

To reconfigure at any time (change provider, model, API key, etc.), delete the config and relaunch:

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

- **Working dir:** Anything the agent writes (code, docs, artifacts) goes to `working_dir/` on your host, mapped to `/app/data` inside the container. Agent can read from here too.
- **Agent state:** Config, memories, skills, and logs are stored in `hermes_data/`, mapped to `/opt/data` inside the container.
- **Git Safety:** Both `working_dir/` and `hermes_data/` contents are gitignored — none of your conversations, credentials, or agent-generated files will be committed to GitHub. Only the empty directory placeholders are tracked.
- **Structure:**
  - `hermes_data/` - Agent workspace (config, memories, skills). Populated at runtime, never committed.
  - `working_dir/` - Agent-generated files. Populated at runtime, never committed.
  - `docker-compose.yml` - Container orchestration.

---

## Using Podman

This setup is compatible with Podman on **Linux** with no changes — `host-gateway` resolves correctly and `host.docker.internal` works as expected.

On **macOS**, Podman runs inside a Linux VM, so `host-gateway` resolves to the VM rather than your Mac. Replace `host.docker.internal` with `host.containers.internal` in `docker-compose.yml`:

```yaml
extra_hosts:
  - "host.containers.internal:host-gateway"
```

And update `OPENAI_BASE_URL` accordingly:

```yaml
environment:
  - OPENAI_BASE_URL=http://host.containers.internal:1234/v1
```

Then run with:

```bash
podman compose run --rm hermes
```

---

## Recommended Local Models

`Qwen2.5-Coder-7B-Instruct` works for code-heavy tasks but is a code-completion model — not optimized for agentic instruction following, planning, or multi-step tool use. Here are better options:

### Smallest but not strongest

| Model                     | Size (Q4_K_M) | Notes                                                                                                                                                              |
| ------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Hermes-3-Llama-3.1-8B** | ~4.9 GB       | Made by Nous Research (same team as Hermes Agent). Trained specifically on the Hermes prompt/function-calling schema — best alignment with this agent's internals. |
| **Qwen2.5-7B-Instruct**   | ~4.7 GB       | The general-instruction variant of Qwen2.5 — noticeably better at following complex instructions than the Coder variant.                                           |

### Better quality (14–24B, needs ~16GB+ RAM)

| Model                     | Size (Q4_K_M) | Notes                                                    |
| ------------------------- | ------------- | -------------------------------------------------------- |
| **Phi-4-14B**             | ~8.5 GB       | Strong reasoning and instruction following for its size. |
| **Qwen2.5-14B-Instruct**  | ~8.9 GB       | Solid all-around upgrade from the 7B.                    |
| **Mistral-Small-3.1-24B** | ~14 GB        | Very capable at tool use and multi-step tasks.           |

### Best local quality (32B+, needs ~32GB+ RAM)

| Model                    | Size (Q4_K_M) | Notes                                         |
| ------------------------ | ------------- | --------------------------------------------- |
| **Qwen2.5-32B-Instruct** | ~19 GB        | Significant jump in agentic task performance. |

**Start with Hermes-3-Llama-3.1-8B** — it's the same size as Qwen2.5-Coder-7B, purpose-built for agentic workflows, and directly aligned with Hermes Agent's prompt format. All models are available as GGUF on HuggingFace and loadable in LM Studio.

Remember to set a **Context Length ≥ 17000** (32768 recommended) in LM Studio for whichever model you use — the Hermes system prompt alone is ~17K tokens.
