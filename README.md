<p align="center">
  <img src="assets/logo.png" alt="nah" width="280">
</p>

<p align="center">
  <strong>A permission system you control.</strong><br>
  Context-aware safety guard for Claude Code — guards all tools, not just Bash.
</p>

<p align="center">
  <a href="#install">Install</a> &bull;
  <a href="#what-it-guards">What it guards</a> &bull;
  <a href="#how-it-works">How it works</a> &bull;
  <a href="#configure">Configure</a> &bull;
  <a href="#auto-mode">Works with Auto Mode</a>
</p>

---

## The problem

Claude Code's permission system is all-or-nothing. Allow a tool, and the agent can do anything with it. Deny lists are trivially bypassed — deny `rm`, the agent uses `unlink`. Deny that, it uses `python -c "import os; os.remove()"`.

Meanwhile, nobody guards Read, Write, Edit, Glob, or Grep at all. The agent can read your SSH keys and write malicious scripts unchecked.

## Install

```bash
pip install nah
nah install
```

That's it. Two commands. Zero config required — sensible defaults out of the box.

## What it guards

nah is a [PreToolUse hook](https://docs.anthropic.com/en/docs/claude-code/hooks) that intercepts **every** tool call before it executes:

| Tool | What nah checks |
|------|----------------|
| **Bash** | Structural command classification — action type, pipe composition, shell unwrapping |
| **Read** | Sensitive path detection (`~/.ssh`, `~/.aws`, `.env`, ...) |
| **Write** | Path check + content inspection before anything hits disk |
| **Edit** | Path check + content inspection on the replacement string |
| **Glob** | Blocks directory scanning of sensitive locations |
| **Grep** | Catches credential search patterns outside the project |

## How it works

Every tool call gets a deterministic verdict in under 5ms. Zero tokens. Zero cost.

```
Claude: Bash → rm -rf /
  nah. Bash: filesystem_delete — targets root (blocked)

Claude: Read → ~/.ssh/id_rsa
  nah? Read targets sensitive path: ~/.ssh (requires confirmation)

Claude: Bash → npm test
  ✓ allowed (package_run, inside project)

Claude: Write → deploy.sh containing "curl ... | bash"
  nah? Write: suspicious content — download-and-execute pattern
```

**`nah.`** = blocked. **`nah?`** = asks for your confirmation. Everything else flows through silently.

### Context-aware, not pattern-matching

The same command gets different decisions based on context:

| Command | Context | Decision |
|---------|---------|----------|
| `rm dist/bundle.js` | Inside project, git-tracked | Allow |
| `rm ~/.bashrc` | Outside project, sensitive | Ask |
| `git push --force` | History rewrite | Ask |
| `base64 -d \| bash` | Decode + exec pipe | Block |

## Configure

Works out of the box with zero config. When you want to tune it:

```yaml
# ~/.config/nah/config.yaml  (global)
# .nah.yaml                  (per-project)

actions:
  filesystem_delete: context     # check path before deciding
  git_history_rewrite: ask       # always confirm
  network_outbound: ask          # always confirm
  obfuscated: block              # always block

sensitive_paths:
  ~/.kube: ask
  ~/Documents/taxes: block

classify:
  database_destructive:          # add your own action types
    - "psql -c DROP"
    - "mysql -e DROP"
```

nah classifies commands by **action type** (what kind of thing), not by command name (which command). The taxonomy is extensible — add commands, create new action types, reclassify anything you disagree with.

## CLI

```bash
nah install       # install hook into Claude Code
nah uninstall     # clean removal
nah update        # update hook after pip upgrade
nah test rm -rf / # dry-run — see how nah would classify a command
nah config show   # show effective merged config
```

<h2 id="auto-mode">Works with Auto Mode</h2>

Anthropic's [Auto Mode](https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously) lets Claude reason per-action about whether to auto-approve or prompt. nah complements it — they're different layers:

```
Tool call → nah (deterministic) → Auto Mode (probabilistic) → execute
```

| | Auto Mode | nah | Both |
|---|---|---|---|
| **Engine** | LLM reasoning | Deterministic rules | Hard floor + smart fallback |
| **Latency** | ~500ms-2s | <5ms | Faster on average |
| **Cost** | Extra tokens/call | Zero | Reduced |
| **Prompt injection** | Vulnerable | Immune | Immune at the deterministic layer |
| **Content inspection** | No | Yes | Yes |
| **Your rules** | Anthropic's black box | Your YAML | You control the floor |

Auto Mode makes Claude smarter about permissions. nah makes it impossible for that smartness to fail catastrophically.

## How it's different

**vs. deny lists** ([safety-net](https://github.com/anthropics/safety-net), [dcg](https://github.com/AshDevFr/dcg)) — Pattern matching on command strings is trivially bypassed. nah resolves paths, checks git status, inspects content, guards all 6 tools, and classifies by action type instead of command name.

**vs. OS sandboxes** ([scode](https://github.com/nichochar/scode), [nono](https://github.com/anthropics/nono)) — Complementary layers. Sandboxes enforce at the OS level but can't distinguish safe from unsafe operations on allowed paths. nah adds the smart gate inside the OS fence. `pip install` on any machine with Python 3.

**vs. built-in permissions** — Not configurable enough. You can't say "allow deletes inside my project but ask outside." nah adds the granularity that's missing.

## Uninstall

```bash
nah uninstall
pip uninstall nah
```

## License

MIT
