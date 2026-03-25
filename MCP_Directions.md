# OpenClaw Workshop: NAI-Routed Hugging Face MCP

## Getting Started

### 1. Deploy the Launchable

Go to [Brev Deploy](https://brev.nvidia.com/launchable/deploy/now?launchableID=env-3Azt0aYgVNFEuz7opyx3gscmowS) and click **Deploy Launchable**.

### 2. Open Code Server

Once deployed, open the **Code Server** tab. The NeMoClaw automated installer will start immediately.

> **This startup takes approximately 15 minutes.** You'll see a progress sequence in the terminal — let it run to completion.

The installer runs through three main phases:

#### Phase 1 — Node.js

Installs Node.js via nvm (needed for the MCP toolchain).

```text
[1/3] Node.js
  ──────────────────────────────────────────────────
[INFO]  Node.js not found — installing via nvm…
  ✓  Installing nvm...
  ✓  Installing Node.js 22...
```

#### Phase 2 — NemoClaw CLI

Builds and links the NemoClaw CLI from source.

```text
[2/3] NemoClaw CLI
  ──────────────────────────────────────────────────
  ✓  Preparing OpenClaw package
  ✓  Installing NemoClaw dependencies
  ✓  Building NemoClaw plugin
  ✓  Linking NemoClaw CLI
```

#### Phase 3 — Onboarding (longest step)

This is the bulk of the wait. The onboarding wizard runs seven sub-steps:

| Sub-step | What happens |
|----------|-------------|
| **[1/7] Preflight checks** | Verifies Docker, ports, and GPU availability |
| **[2/7] Configuring inference** | You enter the inference endpoint (see step 3 below) |
| **[3/7] Starting OpenShell gateway** | Downloads and launches the gateway container |
| **[4/7] Setting up inference provider** | Configures the model route |
| **[5/7] Creating sandbox** | Builds and pushes the sandbox image (~1 GB) — **this is the slowest part** |
| **[6/7] Setting up OpenClaw** | Launches the OpenClaw gateway inside the sandbox |
| **[7/7] Policy presets** | Applies default network policies (accept the suggested presets) |

### 3. Enter the inference endpoint

During **[2/7] Configuring inference**, the installer will prompt you interactively. Enter:

1. **Choose option:** `3` (Other OpenAI-compatible endpoint)
2. **Base URL:** `https://passage-arthritis-sessions-lawsuit.trycloudflare.com/enterpriseai/v1`
3. **API key:** (will be provided)
4. **Model name:** `gpt-oss-120b`

After you enter these, the rest of the install continues automatically.

### 4. Wait for the install to finish

When complete, you'll see a summary like this:

```text
  ──────────────────────────────────────────────────
  Sandbox      my-assistant (Landlock + seccomp + netns)
  Model        gpt-oss-120b (Other OpenAI-compatible endpoint)
  ──────────────────────────────────────────────────
```

Followed by the OpenClaw connection URL:

```text
OpenClaw connection details
  Sandbox: my-assistant
  URL: https://openclaw0-XXXXX.brevlab.com#token=...
```

### 5. Open the chat UI

Copy the full URL (including the `#token=...` part) and paste it into your browser to open the OpenClaw chat.

### 6. Run the MCP setup commands below

Run the following steps in the **Code Server terminal**, then go back to the chat UI to test.

---

You can safely run the steps more than once.

## What this gives you

Your OpenClaw chat will answer Hugging Face questions using the NAI-routed MCP endpoint.

Good demo questions:

- search Transformers docs
- find top Meta Llama models
- find recent ML papers

---

## Step 1: Set up SSH and create a writable config

This adds an SSH alias so you can reach the sandbox easily, then copies the OpenClaw config to a writable location and restarts the gateway using it.

```bash
openshell sandbox ssh-config my-assistant >> ~/.ssh/config
ssh openshell-my-assistant 'mkdir -p /sandbox/config && rm -f /sandbox/config/openclaw.json && cp /sandbox/.openclaw/openclaw.json /sandbox/config/openclaw.json && chmod 600 /sandbox/config/openclaw.json && openclaw gateway stop >/dev/null 2>&1 || true && OPENCLAW_CONFIG_PATH=/sandbox/config/openclaw.json nohup openclaw gateway run > /tmp/gateway.log 2>&1 &'
```

Check it:

```bash
ssh openshell-my-assistant 'test -f /sandbox/config/openclaw.json && ss -ltnp 2>/dev/null | grep 18789 && echo ready'
```

You want to see:

```text
LISTEN ... 127.0.0.1:18789 ...
LISTEN ... [::1]:18789 ...
ready
```

## Step 2: Prepare workspace for workshop

This creates the memory files and simplified persona files that the chat expects on startup. Without these, the model gets stuck trying to read files that don't exist and never replies.

```bash
ssh openshell-my-assistant 'python3 - <<'\''PY'\''
from pathlib import Path
from datetime import date, timedelta

root = Path("/sandbox/.openclaw/workspace")
(root / "memory").mkdir(parents=True, exist_ok=True)
(root / "MEMORY.md").touch()
(root / "memory" / f"{date.today().isoformat()}.md").touch()
(root / "memory" / f"{(date.today() - timedelta(days=1)).isoformat()}.md").touch()
(root / "BOOTSTRAP.md").unlink(missing_ok=True)

for name in ["TOOLS.md", "HEARTBEAT.md"]:
    p = root / name
    if p.exists():
        p.unlink()

(root / "SOUL.md").write_text("# Workshop Assistant\n\nYou are a helpful workshop assistant. Be concise.\n")
(root / "IDENTITY.md").write_text("# Identity\n\nName: Workshop Bot\n")
(root / "USER.md").write_text("# User\n\nWorkshop attendee.\n")
(root / "AGENTS.md").write_text("# AGENTS.md\n\nWorkshop mode. No startup file reads required. Just reply to the user.\n")

print("workspace-ready")
PY'
```

You want to see:

```text
workspace-ready
```

## Step 3: Patch the sandbox network policy

This allows the sandbox to reach the Hugging Face MCP endpoint and Hugging Face websites. The sandbox blocks all outbound traffic by default, so without this the MCP calls would be denied.

```bash
openshell policy get --full my-assistant > policy-full.txt
python3 - <<'PY'
from pathlib import Path

raw = Path("policy-full.txt").read_text()
yaml_text = raw.split("---", 1)[1].strip() if "---" in raw else raw.strip()

entry = """
  huggingface_mcp_route:
    name: huggingface_mcp_route
    endpoints:
      - host: passage-arthritis-sessions-lawsuit.trycloudflare.com
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/**" }
          - allow: { method: POST, path: "/**" }
      - host: hf.co
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/**" }
          - allow: { method: POST, path: "/**" }
      - host: huggingface.co
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/**" }
          - allow: { method: POST, path: "/**" }
    binaries:
      - { path: /usr/local/bin/node }
      - { path: /usr/bin/node }
      - { path: /usr/bin/curl }
""".rstrip("\n")

if "network_policies:" not in yaml_text:
    yaml_text = yaml_text.rstrip() + "\n\nnetwork_policies:\n" + entry + "\n"
elif "huggingface_mcp_route:" not in yaml_text:
    yaml_text = yaml_text.replace("network_policies:\n", "network_policies:\n" + entry + "\n")

Path("policy.yaml").write_text(yaml_text)
print("Wrote policy.yaml")
PY
openshell policy set --policy ./policy.yaml --wait my-assistant
```

You want to see:

```text
Wrote policy.yaml
✓ Policy version ... submitted
✓ Policy version ... loaded
```

## Step 4: Upload the MCP server config

This creates and uploads the `mcporter.json` file that tells the sandbox how to connect to the NAI-routed Hugging Face MCP server, including the endpoint URL and auth token.

Set the API key (will be provided):

```bash
export HF_MCP_KEY="your-key-here"
```

Then run:

```bash
mkdir -p config
python3 - <<'PY'
import os
from pathlib import Path

key = os.environ.get("HF_MCP_KEY")
if not key:
    raise SystemExit("Set HF_MCP_KEY first: export HF_MCP_KEY='your-key'")

Path("config").mkdir(exist_ok=True)
Path("config/mcporter.json").write_text(
    f'{{"mcpServers":{{"huggingface-nai":{{"baseUrl":"https://passage-arthritis-sessions-lawsuit.trycloudflare.com/enterpriseai/mcp/huggingface","headers":{{"Authorization":"Bearer {key}"}}}}}},"imports":[]}}'
)
print("Wrote config/mcporter.json")
PY
openshell sandbox upload my-assistant ./config /sandbox/config
```

You want to see:

```text
Wrote config/mcporter.json
Uploading ./config -> sandbox:/sandbox/config
✓ Upload complete
```

## Step 5: Install mcporter inside the sandbox

The skill commands use `npx mcporter call ...` to invoke MCP tools. Since the sandbox blocks most outbound traffic, `npx` can't download mcporter on the fly — it needs to be pre-installed.

```bash
ssh openshell-my-assistant 'cd /sandbox && npm install mcporter && echo mcporter-ready'
```

You want to see:

```text
added ... packages ...
mcporter-ready
```

## Step 6: Install the Hugging Face skill

This installs a skill file that teaches the OpenClaw agent how to call the Hugging Face MCP tools. When you ask about Hugging Face topics in chat, the agent reads this skill and knows which command to run.

```bash
mkdir -p skill/huggingface
python3 - <<'PY'
from pathlib import Path

Path("skill/huggingface").mkdir(parents=True, exist_ok=True)
Path("skill/huggingface/SKILL.md").write_text(
    """---
name: huggingface
description: "USE THIS SKILL for questions about Hugging Face docs, models, datasets, Spaces, and ML papers."
---

# Hugging Face MCP

Run these commands from /sandbox using exec.

## Docs search

exec: cd /sandbox && npx mcporter call 'huggingface-nai.nai-2e14096a-5779-434a-8fb8-99__hf_doc_search' query='<user question>'

## Models search

exec: cd /sandbox && npx mcporter call 'huggingface-nai.nai-2e14096a-5779-434a-8fb8-99__hub_repo_search' query='<query>' repo_types:='["model"]' limit=3

## Paper search

exec: cd /sandbox && npx mcporter call 'huggingface-nai.nai-2e14096a-5779-434a-8fb8-99__paper_search' query='<query>' results_limit=3 concise_only=true

## Rules

- Pick the right command above based on the user question.
- Run the command with exec. Do NOT list tools first.
- Include links from the results in your answer.
- Keep answers short and grounded in the output.
"""
)
print("Wrote skill/huggingface/SKILL.md")
PY
openshell sandbox upload my-assistant ./skill/huggingface /sandbox/.agents/skills
ssh openshell-my-assistant 'mkdir -p /sandbox/.agents/skills/huggingface ~/.openclaw/skills/huggingface ~/.claude/skills/huggingface /sandbox/.openclaw-data/skills/huggingface && cp /sandbox/.agents/skills/SKILL.md /sandbox/.agents/skills/huggingface/SKILL.md && cp /sandbox/.agents/skills/huggingface/SKILL.md ~/.openclaw/skills/huggingface/SKILL.md && cp /sandbox/.agents/skills/huggingface/SKILL.md ~/.claude/skills/huggingface/SKILL.md && cp /sandbox/.agents/skills/huggingface/SKILL.md /sandbox/.openclaw-data/skills/huggingface/SKILL.md && echo ready'
```

You want to see:

```text
Wrote skill/huggingface/SKILL.md
Uploading ./skill/huggingface -> sandbox:/sandbox/.agents/skills
✓ Upload complete
ready
```

## Step 7: Restart the gateway

This restarts the OpenClaw gateway so it picks up the new config, policy, and skill files. The gateway is what connects the chat UI to the model and tools.

```bash
ssh openshell-my-assistant 'openclaw gateway stop >/dev/null 2>&1 || true && OPENCLAW_CONFIG_PATH=/sandbox/config/openclaw.json nohup openclaw gateway run > /tmp/gateway.log 2>&1 &'
sleep 5
ssh openshell-my-assistant 'ss -ltnp 2>/dev/null | grep 18789'
```

You want to see:

```text
LISTEN ... 127.0.0.1:18789 ...
LISTEN ... [::1]:18789 ...
```

## Step 8: Test it

Everything is set up. Now test it in the chat UI.

1. Go back to the chat UI in your browser.
2. Refresh the page.
3. Type `/new`.
4. Try these prompts:

```text
Use Hugging Face MCP to explain text generation in Transformers.
```

```text
Use Hugging Face MCP to find the top Meta Llama instruct models.
```

```text
Use Hugging Face MCP to find recent papers about diffusion transformers.
```

---

## Trouble Help

### No response in chat

Run this:

```bash
ssh openshell-my-assistant 'python3 - <<'\''PY'\''
from pathlib import Path
from datetime import date, timedelta

root = Path("/sandbox/.openclaw/workspace")
(root / "memory").mkdir(parents=True, exist_ok=True)
(root / "MEMORY.md").touch()
(root / "memory" / f"{date.today().isoformat()}.md").touch()
(root / "memory" / f"{(date.today() - timedelta(days=1)).isoformat()}.md").touch()
(root / "BOOTSTRAP.md").unlink(missing_ok=True)

for name in ["TOOLS.md", "HEARTBEAT.md"]:
    p = root / name
    if p.exists():
        p.unlink()

(root / "SOUL.md").write_text("# Workshop Assistant\n\nYou are a helpful workshop assistant. Be concise.\n")
(root / "IDENTITY.md").write_text("# Identity\n\nName: Workshop Bot\n")
(root / "USER.md").write_text("# User\n\nWorkshop attendee.\n")
(root / "AGENTS.md").write_text("# AGENTS.md\n\nWorkshop mode. No startup file reads required. Just reply to the user.\n")

print("workspace-ready")
PY'
ssh openshell-my-assistant 'openclaw gateway stop >/dev/null 2>&1 || true && OPENCLAW_CONFIG_PATH=/sandbox/config/openclaw.json nohup openclaw gateway run > /tmp/gateway.log 2>&1 &'
```

Then refresh the browser, type `/new`, and try again.

### `Bad gateway`

```bash
ssh openshell-my-assistant 'openclaw gateway stop >/dev/null 2>&1 || true && OPENCLAW_CONFIG_PATH=/sandbox/config/openclaw.json nohup openclaw gateway run > /tmp/gateway.log 2>&1 &'
```

Wait a few seconds. Refresh the browser.

### Skill does not trigger

Use this exact wording:

```text
Use Hugging Face MCP to explain text generation in Transformers.
```
