# Set up the SonarQube plugin for Cursor

## TL;DR overview

- The SonarQube plugin for Cursor brings issue scanning, quality gate checks, code coverage, dependency risks, secrets scanning, Agentic Analysis, and Context Augmentation into Cursor through dedicated `sonar-*` skills, the SonarQube MCP Server, and a `sonar-integrate` setup command that installs the plugin's hooks and rules.  
- This setup fits Cursor users who want SonarQube results in their chat session instead of waiting on a PR or CI run.  
- SonarQube findings and fixes inside Cursor compress the development cycle from prompting and PR-time review to analysis, remediation, and verification in a single chat session.  
- Setup installs the SonarQube plugin from the Cursor marketplace, runs `sonar-integrate` to wire authentication, the MCP server, three secrets-scanning hooks, an Agentic Analysis rule, and a Context Augmentation skill into the project, then enables the MCP in Cursor Settings.

This blueprint configures the [SonarQube plugin](https://github.com/SonarSource/sonarqube-agent-plugins) for [Cursor](https://cursor.com/) so that issue scanning, quality gates, coverage, dependency risks, secrets detection, [Agentic Analysis](https://www.sonarsource.com/products/sonarqube/agentic-analysis/), and [Context Augmentation](https://www.sonarsource.com/products/context-augmentation/) are accessible from inside the same Cursor chat sessions that write your code. The plugin contributes a pre-defined `sonar-*` skill set, the [SonarQube MCP Server](https://www.sonarsource.com/products/sonarqube/mcp-server/), and a `sonar-integrate` setup command that wires authentication and agent hooks into your project. Under the hood, the [SonarQube CLI](https://cli.sonarqube.com/) is the runtime the plugin's setup command depends on. Our step-by-step walkthrough follows the Python-based [AWS CLI](https://github.com/aws/aws-cli), but the steps apply to any project written in any [SonarQube-supported language](https://www.sonarsource.com/knowledge/languages/).

## When to use this

- You drive Cursor as your primary AI coding tool and want SonarQube capabilities available without leaving the chat.  
- You want issue surfacing, rule-driven fixes, and architectural context inside the same session that writes your code, not waiting on the next CI run or a PR-time review.

## What you'll achieve

- SonarQube plugin installed in Cursor with the SonarQube MCP Server and the `sonar-*` skill set available from the chat box.  
- A `.cursor/` directory wired with the MCP registration, three secrets-scanning hooks (`beforeSubmitPrompt`, `preToolUse`, `beforeReadFile`), an Agentic Analysis rule, and a Context Augmentation skill at `.agents/skills/sonar-context-augmentation/SKILL.md`.  
- A working loop where Cursor writes code, the plugin runs Agentic Analysis automatically at end-of-turn, surfaces findings, and applies rule-driven fixes inline.

## Architecture

![Architecture diagram](screenshots/cursor-plugin-architecture.png)

The SonarQube plugin contributes everything Cursor needs to talk to SonarQube: the SonarQube MCP server, the `sonar-*` skill set, three secrets-scanning hooks, an Agentic Analysis Cursor rule, and a Context Augmentation skill. The plugin's `sonar-integrate` setup command writes those artifacts into your project's `.cursor/` directory and the Context Augmentation skill into `.agents/skills/`. Under the hood, the SonarQube CLI is the runtime; the MCP server runs as the mcp/sonarqube container, launched by `sonar run mcp`, every hook script shells out to `sonar hook cursor-*`, the Agentic Analysis loop calls `sonar analyze agentic`, and Context Augmentation calls `sonar context guidelines get`.

## Prerequisites

- [**Cursor**](https://cursor.com/) installed and authenticated  
- [**SonarQube Cloud**](https://www.sonarsource.com/products/sonarqube/cloud/) **account** or [**SonarQube Server**](https://www.sonarsource.com/products/sonarqube/server/) with the demo project on board  
- **Docker, Podman, or Nerdctl** running; the MCP server runs as a container  
- [**SonarQube CLI**](https://cli.sonarqube.com/) installed (v1.1.0 or later)  
- **(Optional, but preferred)** a `sonar-project.properties` file pointing to your project  
- **(Optional) Agentic Analysis** and **Context Augmentation** enabled at your SonarQube Cloud organization level (**Administration** → **Agent-Centric Development**)

**Demo project:** Fork [`aws/aws-cli`](https://github.com/aws/aws-cli) to your GitHub account, import it into SonarQube Cloud with CI-based analysis enabled, and clone it locally.

## Step 1 — Install the SonarQube plugin from the Cursor marketplace

Open the [Cursor marketplace](https://cursor.com/marketplace/sonarsource), search for the `SonarQube` listing, and click **Get**.

Sign in to your Cursor account if prompted, open Cursor, and navigate to **Settings → Plugins** to confirm that the installation succeeded.

The plugin contributes the `sonar-*` skills you'll use in the next steps but does not bring up the MCP server or install the hooks. That's the job of `sonar-integrate`, covered next.

## Step 2 — Configure the integration

You can invoke a skill directly in the chat box by selecting it from the **Skills** directory (click **\+** then **Skills** for the applied */slash* command), by **@**\-prefacing it with **sonarqube**, or with natural language (e.g. "List my SonarQube projects").

Within Cursor, select a **New Agent**, open your project (`aws-cli` for the demo), and type `sonar-integrate` into the chat box:

```shell
sonarqube sonar-integrate
```

The skill first checks if SonarQube CLI is installed. If not, you'll be prompted to allow Cursor to install it. If `sonar` is already on your `PATH`, Cursor runs `sonar self-update` instead.

You'll then be prompted to choose your SonarQube connection type (SonarQube Cloud — EU by default) and enter your organization key (if it’s not already specified in a `sonar-project.properties` file).

For this walkthrough, specify the integration scope as **Current project only**.

Then authenticate with your organization key:

```shell
sonar auth login -o <your-organization-key-here>
```

![][image7]

The subsequent `sonar auth login` flow opens your browser, runs OAuth, and stores the resulting token in your system keychain. Every downstream command (including the MCP server and hook scripts) reuses that session.

When the browser opens, click **Allow connection** and confirm that the authentication succeeded:  
![][image8]  
![][image9]

Inform Cursor once you’ve authenticated for confirmation that the integration is ready.

To recap, the plugin's setup command discovers your project and writes:

- `.cursor/mcp.json` registering the `sonarqube` MCP server  
- `.cursor/hooks.json` registering three secrets-scanning hooks (`beforeSubmitPrompt`, `preToolUse`, `beforeReadFile`) plus their hook scripts under `.cursor/hooks/sonar-secrets/build-scripts/`  
- `.cursor/rules/sonar-agentic-analysis.mdc` — an always-applied Cursor rule that runs Agentic Analysis at end-of-turn  
- `.agents/skills/sonar-context-augmentation/SKILL.md` — the Context Augmentation skill  
- Telemetry configuration

`sonar-integrate` is idempotent, so re-running it simply reports `already installed` for artifacts that haven't changed.

## Step 3 — Enable the SonarQube MCP Server

Cursor does not auto-enable MCP servers after `sonar-integrate`. Open **Settings**, navigate to the **Tools & MCPs** tab, and toggle `sonarqube` on. If necessary, reload the MCP server.

Once enabled, click into **tools enabled** for a list of the tools exposed by the SonarQube MCP Server.

## Step 4 — Invoke the plugin skills

Confirm the plugin's `sonar-*` skills are wired by listing your SonarQube projects from the Cursor chat.

```
sonarqube list my projects
```

Then list open issues in the `aws-cli` project:

```
sonarqube list the top 10 issues in our aws-cli project
```

Lastly (and optionally), in natural language, invoke Context Augmentation by inquiring about our project’s architecture before instructing Cursor to write code:

```
sonarqube what is the current architecture of the project that we're working within?
```

This skill is always invoked by the agent on the first prompt, kicking in automatically before code generation so future prompts that write or edit code are informed by SonarQube's view and match project conventions.

## Step 5 — Watch the secrets hook block a prompt

The `beforeSubmitPrompt` hook installed in Step 2 sits between you and the model. Every prompt is scanned for credentials before it's submitted; if a token pattern is detected, the prompt is blocked outright.

Paste the following (fake) GitHub personal access token into your prompt:

```
Can you push a commit using my token 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?
```

The hook detects the GitHub PAT pattern and surfaces a denial; Cursor never receives the prompt.

The same scanner sits in front of file reads via the `preToolUse(Read)` and `beforeReadFile(Read|TabRead)` hooks. The plugin's design choice here is important: when either hook detects a secret, it returns a deny, but the deny on its own doesn't keep Cursor away from the file. Cursor's agent treats the denial as a soft failure and tries another tool to reach the file. What actually locks the file out is `.cursorignore`: the plugin appends the file's path to `.cursorignore` before returning the deny, and Cursor's platform then enforces that ignore list natively on every subsequent attempt. To restore access after fixing the leak, rotate the credential, remove it from the file, and delete the entry from `.cursorignore`.

## Step 6 — (Optional) write code and watch Agentic Analysis fire automatically

The Agentic Analysis Cursor rule installed in Step 2 tells the agent to run `sonar analyze agentic --file <path>` for every file it touches before ending the turn. Ask Cursor to write a small module:

```
Add s3_helper.py at the project root with an upload_with_retry(bucket, key, file_path, max_retries=3) function. Retry on transient errors, log each attempt, and raise a clear error if all retries fail. Include a TODO for adding metric emission later.
```

Cursor writes the file with a `boto3.client('s3').upload_file(...)` call inside a retry loop, an `S3UploadError` for terminal failures, structured logging on each attempt, and the TODO. Before ending the turn, the rule kicks in: the agent runs Agentic Analysis, surfaces two findings (`python:S1135` (the TODO comment) and `python:S7608` (missing `ExpectedBucketOwner` on the S3 upload)),  applies the rule-driven fix for `python:S7608` automatically, and re-runs Agentic Analysis to confirm the rule is resolved before returning control to you.


`python:S7608` is the catch worth investigating. It’s a High-severity security rule (CWE-441, CWE-693) that arises because an S3 `upload_file` call without `ExpectedBucketOwner` can silently succeed against a bucket whose ownership has changed out from under the caller. The TODO finding (`python:S1135`) is the one Cursor was explicitly asked to add in Step 7, so the agent leaves it open by design.

## Verify the setup

By the end of Step 3, you have a working SonarQube \+ Cursor integration. The end state across your system should reflect:

- The SonarQube plugin installed from the Cursor marketplace  
- `sonar --version` reports 1.1.0 or later and `sonar auth status` returns authenticated  
- `.cursor/mcp.json`, `.cursor/hooks.json`, `.cursor/rules/sonar-agentic-analysis.mdc`, and the three hook scripts under `.cursor/hooks/sonar-secrets/build-scripts/`  
- `.agents/skills/sonar-context-augmentation/SKILL.md` in place  
- `sonarqube` toggled on under **Settings → Tools & MCPs**

If anything drifts (the MCP stops appearing, hooks stop firing, or skills return empty), re-run `sonar-integrate` as it's idempotent.

## What to know

- **Cursor does not auto-enable MCP servers after `sonar-integrate`.** Open **Settings** → **Tools & MCPs** and toggle `sonarqube` on. Without this step, every skill that routes through the MCP server will fail silently.  
- The integration uses `sonar run mcp`, not a raw Docker invocation. The SonarQube CLI handles container runtime detection (Docker, Podman, Nerdctl) and the keychain handoff under the hood.  
- If `/mcp` reports `sonarqube` as failed or not started, first confirm that Docker is running, then restart the Cursor session. Lack of a container runtime is a common cause for such failures.  
- Sonar Context Augmentation is currently only supported on SonarQube Cloud.  
- If you also use the [SonarQube for IDE extension](https://www.sonarsource.com/resources/library/sq-ide-plug-in-for-cursor/) inside Cursor, the two are complementary. The extension drives real-time editor feedback through Connected Mode; this plugin drives the in-chat agent loop.

## Next steps

- Set up the SonarQube Plugin for [Claude Code](https://www.sonarsource.com/resources/library/set-up-the-sonarqube-plugin-for-claude-code/), [GitHub Copilot CLI](https://www.sonarsource.com/resources/library/set-up-the-sonarqube-plugin-for-github-copilot-cli/), and [more](https://github.com/SonarSource/sonarqube-agent-plugins)  
- Configure the [SonarQube for IDE extension in Cursor](https://www.sonarsource.com/resources/library/sq-ide-plug-in-for-cursor/) for real-time editor feedback alongside the plugin  
- Familiarize yourself with [SonarQube CLI](https://cli.sonarqube.com/) [commands](https://docs.sonarsource.com/sonarqube-cli/using/commands)  
- Dive deeper into the [SonarQube MCP Server docs](https://docs.sonarsource.com/sonarqube-mcp-server/about-the-mcp-server)  
- Connect the Guide, Verify, and Solve dots with Sonar's [Agent-Centric Development Cycle](https://www.sonarsource.com/agent-centric-development/) framing