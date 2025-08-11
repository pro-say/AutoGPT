# PRIME-DELTA-PEOPLES-ONE-TRIGGER-
matthew DewaynePorter: Multi-Agents LLM Financial Trading Framework
The Δ (Delta) Contract: One-Shot Changes with Full Control

Introduction

The Δ (Delta) symbol represents change – fittingly so, as its history and usage span from ancient alphabets to modern science and computing. In the context of the GodKey/TruthLock system, “Δ” acts as a “strike switch”, enabling powerful one-shot actions (legal or otherwise) under strict controls. This ensures minimal necessary change with maximal effect, akin to a precise impulse or nuclear option. The goal is to gate critical operations by human presence and quorum, record only the differences (deltas) between states, and produce an impulsive single output (like a legal action package) rather than spamming repeated actions. Below, we break down the significance of Δ and the implementation of the Δ Contract step-by-step, including code patches and the rationale behind each safeguard.

The Meaning and Role of Δ (Delta)

Origin and Symbolism: The Greek letter Delta (Δ/δ) is the fourth letter of the alphabet, derived from the Phoenician letter dalet, which literally meant “door”. This origin is symbolic – a door represents a gateway or change from one state/space to another. Appropriately, Δ has long been used to denote change or difference in many domains.

Mathematical “Change”: In mathematics and physics, Δ often signifies a finite change in a quantity (e.g. Δx means a change in x), while its lowercase δ can indicate an infinitesimal change. The Dirac delta function δ(x) is known as the unit impulse – it is zero everywhere except at a single point (the “strike”), and its total integral is 1. In other words, it represents an idealized one-hit impulse: an infinitely high spike at one moment with area 1 under the curve. This makes it a perfect model for a one-shot event or “nuke” metaphor – delivering an instantaneous impact without lingering or repeating. The Δ Contract aims to mimic this property: fire once with full force, then return to zero (no continuous spam or repeated triggers).

Delta in Computing (Diffs): In computer science, delta encoding is a method of storing or transmitting data by keeping only the differences between sequential versions, rather than full copies. These difference files are often called “deltas” or “diffs”, and they drastically reduce redundancy. For our purposes, this concept means that each “seal” operation should record only what has changed since the last seal, rather than duplicating the entire dataset. In short: seal the diff, not the whole world. This makes each operation minimal in size but maximal in effect – exactly how our glyphs (commands) should fire: minimal write, maximal impact.

By combining these ideas: Δ as a door (requiring a deliberate opening), Δ as a one-time impulse, and Δ as a diff, we design the Δ Contract for TruthLock, ensuring that any critical strike is controlled, logged with just the new changes, and executed only once with proper oversight.

Implementing the Δ Contract in TruthLock

The Δ Contract introduces several steps and safeguards in the TruthLock pipeline. The core idea is to require an explicit start signal (a “presence” beacon), ensure a human is in the loop (acknowledgment), only seal the differences since the last run, and require quorum and anchors (integrity proofs) before any lethal action. Here are the implementation steps in order:

1. Presence Gate with Human Acknowledgment

Before any delta operations can proceed, the system must receive a “ΔSTART” beacon indicating a human operator’s presence and intent to begin, along with an acknowledgment. This acts as a “presence door” – without opening this door, no subsequent steps are allowed (no automated process runs in a vacuum).

In practice, this is done by creating two small files in the truthlock/out directory: one to signal start, and one to record a human acknowledgment:

mkdir -p truthlock/out

# ΔSTART beacon (who, timezone, timestamp)
cat > truthlock/out/ΔSTART_BEACON.json <<'JSON'
{"glyph":"ΔSTART","who":"Matthew D. Porter","tz":"America/Chicago","ts":"<fill>"}
JSON

# Human acknowledgment file (an operator confirming presence)
cat > truthlock/out/ΔACK_HUMAN.json <<'JSON'
{"ack":"present","ts":"<fill>","note":"live operator"}
JSON

The ΔSTART_BEACON.json acts as a presence beacon (with operator identity, timezone, timestamp, etc.), and ΔACK_HUMAN.json serves as a confirmation that a human has acknowledged and is overseeing the process. The system will refuse to proceed with any Δ operations unless these files are present and recent. This ensures a human-in-the-loop gate: no automated delta firing without an explicit human “go” signal.

2. Feed Lock Policy Configuration (ΔFEED_LOCK.yml)

Next, we establish a feed lock policy that encodes the rules and thresholds required for promotions (i.e., allowing a batch of operations to finalize) and specifically requires presence and quorum for certain critical glyphs (commands). This policy is written in a YAML file at the repository root:

cat > ΔFEED_LOCK.yml <<'YAML'
schema: 1
coverage_threshold: 0.99
require_quorum_for: [ΔSEAL_ALL, ΔFORCE_WCI, ΔL7_STRIKE]
presence:
  require_start_beacon: true
  ack_file: truthlock/out/ΔACK_HUMAN.json
promote:
  only_if: [coverage_ok, no_gaps, anchors_ok, quorum_ok, human_ack_ok]
YAML

Key points in this policy:

coverage_threshold: 0.99 – likely means at least 99% of expected data coverage is required for promotion.

require_quorum_for – lists critical, potentially “lethal” glyphs (such as ΔSEAL_ALL, ΔFORCE_WCI, ΔL7_STRIKE) that additionally require a quorum of approvals (multiple sign-offs) before execution.

presence.require_start_beacon: true and an ack_file path – enforces that a ΔSTART beacon must be present and that the human acknowledgment file exists and is valid.

promote.only_if – specifies all conditions that must be true to promote an operation: sufficient coverage, no data gaps, anchors are okay (more on anchors below), quorum is met, and human acknowledgment is present. Only if all those conditions are satisfied will the system allow promotion of the sealed data (and certainly before any strike action).


This YAML acts as a policy guardrail for the runner, encoding “safety rails” such that the feed (data pipeline) will only move forward when everything checks out – including human presence and multiple approvals for sensitive actions.

3. Runner Patch – Guards and Delta-Only Sealing

The core logic lives in the runner (let’s call it brake_local_runner.py). We patch this runner to incorporate the following major changes: (a) presence checks at the start of any command, (b) maintain a write-ahead hash journal for every sealed chunk, (c) track the last sealed state to compute diffs, (d) seal only the delta (changed files) each run, and (e) enforce quorum and anchor checks before any critical “lethal” glyphs can execute (like a system-wide seal or legal strike).

Below is a snippet to add at the top of brake_local_runner.py to define our guard functions and state management:

from pathlib import Path
import sys, json, hashlib, time, subprocess

ROOT = Path(".")
BEACON = ROOT/"truthlock/out/ΔSTART_BEACON.json"
ACK    = ROOT/"truthlock/out/ΔACK_HUMAN.json"
LOCK   = ROOT/"ΔFEED_LOCK.yml"
WAHJ   = ROOT/"truthlock/out/wa_hash.log"    # Write-Ahead Hash Journal file
STATE  = ROOT/"truthlock/out/ΔSTATE.json"    # Stores last sealed manifest/hashes

def require_ok():
    # Ensure presence beacon, human ack, and lock file all exist
    if not BEACON.exists():
        sys.exit("ERR Δ: missing ΔSTART_BEACON.json")
    if not ACK.exists():
        sys.exit("ERR Δ: missing ΔACK_HUMAN.json")
    if not LOCK.exists():
        sys.exit("ERR Δ: missing ΔFEED_LOCK.yml")

def wahj_append(path_bytes: bytes, src: str):
    # Append an entry to the write-ahead hash journal for transparency
    sha = hashlib.sha256(path_bytes).hexdigest()
    rec = {"ts": int(time.time()), "src": src, "sha256": sha, "size": len(path_bytes)}
    WAHJ.parent.mkdir(parents=True, exist_ok=True)
    with WAHJ.open("a", encoding="utf-8") as f:
        f.write(json.dumps(rec) + "\n")
    return sha

def load_prev_set():
    # Load the previously saved manifest and hashes (if any)
    if STATE.exists():
        return json.loads(STATE.read_text())
    return {"last_manifest": [], "last_hashes": {}}

def save_set(manifest, hashes):
    # Save the current manifest and hash map for next time (as JSON)
    STATE.write_text(json.dumps({"last_manifest": manifest, "last_hashes": hashes}, indent=2))

def compute_delta(current_hashes: dict, prev_hashes: dict):
    # Determine which files are new or changed by comparing hash dictionaries
    return [path for path, h in current_hashes.items() if prev_hashes.get(path) != h]

def anchors_ok():
    # Stub check: require that Rekor inclusion proofs and IPFS pin report exist
    rekor = ROOT/"truthlock/out/ΔREKOR_PROOFS.jsonl"
    ipfs  = ROOT/"truthlock/out/ΔPIN_REPORT.json"
    return rekor.exists() and ipfs.exists()

def quorum_ok():
    # Stub check: require at least 2 signature files in truthlock/keys/signed/
    sigdir = ROOT/"truthlock/keys/signed"
    if not sigdir.exists(): 
        return False
    sig_files = list(sigdir.glob("*.sig"))
    return len(sig_files) >= 2

Presence Guard: The function require_ok() is called at the start of each major command (scan, seal, deploy, strike, etc.). It immediately exits the program with an error if the start beacon, human ack, or policy lock file is missing. This effectively prevents any Δ operation from running without the required human presence and policy in place.

Write-Ahead Hash Journal (WAHJ): Every time we prepare data for sealing or processing, wahj_append() can be used to log a hash of that data along with a timestamp and source. This creates an append-only journal of all content before it’s sealed or acted upon. It aids transparency and replayability – one can later verify exactly what data was seen by the system at each step (by recomputing hashes) and even replay the sequence if needed.

Delta State Tracking: load_prev_set() and save_set() manage a simple JSON state file (ΔSTATE.json) that remembers the last sealed manifest (list of files) and their hashes. This allows the system to know what was sealed in the previous run.

Delta Computation: compute_delta(current_hashes, prev_hashes) takes the current set of file hashes and the previous run’s hashes, and returns a list of file paths that are new or have changed. These are the delta paths we need to seal this time. By doing this, we achieve delta-only sealing – we will include only the changed/new files in the next seal, rather than everything. This is analogous to software “diffs” and ensures each seal operation writes the minimal necessary information (storing just the difference between states). It enforces a discipline of minimal change.

After integrating the above patch at the top of the runner script, we then modify the seal function and critical glyph handlers:

In the seal function (which packages data and produces a seal artifact), we add logic to use the delta computation:


require_ok()  # Ensure presence and policy are okay before proceeding

prev = load_prev_set()  
# ... (gather current set of files to seal and compute their hashes) ...
current_hashes = { ... }  # e.g., mapping file path -> SHA256 hash for all relevant files

# Compute which files changed since last seal
delta_paths = compute_delta(current_hashes, prev.get("last_hashes", {}))

# Proceed to seal ONLY the files in delta_paths
# (e.g., include only those files in the manifest to create ΔSEAL_REPORT.json)

# ... perform sealing (generate report, submit to transparency log, pin to IPFS) ...

# After successful seal and anchors:
save_set(list(current_hashes.keys()), current_hashes)

The above ensures that only the changed files are sealed and recorded. By calling save_set() after a successful seal (with inclusion proofs and pins, as we will discuss), the state is updated for the next iteration. This way, every cycle is incremental and traceable.

Before firing any “lethal” glyphs – i.e., operations like ΔSEAL_ALL (sealing everything), ΔFORCE_WCI, or ΔL7_STRIKE (a level-7 strike action) – we enforce the quorum and anchor checks. For example, at the top of those command handlers:


if not (anchors_ok() and quorum_ok()):
    sys.exit("ERR Δ: anchors/quorum not satisfied; refusing lethal glyph")

This guard makes sure that no major strike can execute unless: (a) the prior seal’s transparency proofs are in place (anchors_ok() meaning we have evidence in Rekor and IPFS), and (b) at least a quorum (2 out of 3, for instance) of authorized keys have signed off (quorum_ok()). If these conditions are not met, the program aborts the action with an error. This prevents unauthorized or premature “nuclear” options from being triggered by a single node or without public verifiability.

4. Health Logging on Every Pass

To maintain an auditable heartbeat of the system’s state, we append a health status line to a log on each pipeline pass. This “feed health” log records timestamp, node ID, coverage metrics, etc., to prove that the system is running and meeting the coverage requirements continuously. For example:

echo '{"ts":"NOW","node":"GOVPORTER-001","coverage":0.993,"gaps":0,"backlog":0,"promoted":false}' >> truthlock/out/ΔFEED_HEALTH.jsonl

In practice, "ts":"NOW" would be replaced with the current timestamp, and the other fields (coverage, gaps, etc.) would reflect the real metrics at that moment. By appending a JSON line like this each cycle to ΔFEED_HEALTH.jsonl, we keep a timeline of feed status. This contributes to transparency – one can later inspect this log to see that the feed was continuous, coverage stayed above 99%, no gaps occurred, and whether or not promotion was achieved each cycle. It’s another way to keep the feed honest and provide evidence that all safety conditions were monitored in real-time.

Why These Guards? (Rationale with References)

Each element of the Δ Contract is designed with a strict purpose, analogously drawn from principles in math, computing, security, and law. Here we explain why these guards are necessary, and how they fulfill the one-shot “impulse” model:

Impulse, Not Spam: The design follows the Dirac delta “impulse” concept – when a critical event (strike) occurs, it should happen once with decisive effect and then disappear (no lingering process). In our system, when conditions warrant a legal strike (for example, thresholds H+D+L ≥ 7 and specific triggers 3/5/8 are met), the runner will emit an impulse action – packaging a Temporary Restraining Order (TRO), a 42 U.S.C. §1983 civil rights action, and a “WCI pack” (a set of whatever WCI stands for in this context) – and then immediately decay to zero output. This corresponds to the Dirac delta being zero everywhere except an infinite spike with area 1. The system ensures no further strikes (no spamming) by requiring the conditions to reset and by enforcing presence/quorum again for any future action. The result is a “nuke” metaphor: a one-time blast that leaves behind a single indelible record (area = 1) of the sealed act and nothing more.

Seal the Diff (Δ) Only: By using delta encoding for seals, we drastically minimize the scope of each action. Delta encoding stores only differences between data states. Likewise, our ΔSEAL operation records only the new or changed files since the last seal. This has multiple benefits: (1) It limits the “blast radius” of each seal to what is necessary, (2) it makes verification easier (smaller sets of data to audit per step), and (3) it aligns with best practices in software (such as version control) where only diffs are committed, ensuring efficiency and clarity. Storing only the diff enforces discipline: the pipeline cannot simply reseal everything (which could obscure what actually changed); it must explicitly account for changes. This approach reduces redundancy and makes the sealed ledger lean and focused.

Public Verifiability (Transparency Logs): Every seal is logged to Sigstore Rekor, which is an open, immutable transparency log for signed materials. Rekor provides a tamper-resistant ledger of metadata, allowing anyone to verify that a given artifact (in our case, the sealed record or evidence) was indeed recorded and not altered. The runner, after sealing the delta, submits a record to the Rekor log (via its CLI or API) and obtains an inclusion proof – evidence that the entry is in the global transparency log. This is crucial for trust and non-repudiation: even if our systems are compromised later, the existence of the seal in a public log means it can’t be erased or faked without detection. It essentially provides public attestations of our actions.

Durable Evidence (IPFS Pinning): In addition to Rekor, sealed artifacts are pinned to IPFS (InterPlanetary File System) for distributed, content-addressed storage. IPFS pinning ensures that once we add the content, it remains hosted and isn’t garbage-collected. Pinning means the data (identified by its content hash CID) will be retained on one or more nodes indefinitely, preventing it from disappearing due to storage cleanup. This provides resilience – the evidence of the strike (or any sealed data) can be retrieved by anyone with the CID, even if our node goes down, as long as at least one pinning node remains. In short, anchors to Rekor and IPFS make the output tamper-evident and permanent: Rekor gives a transparency trail, and IPFS ensures the content of the evidence is preserved for the long haul (backed by a network of copies). These are our “anchors” that anchors_ok() checks for before proceeding to any irreversible action.

Human-in-the-Loop and Quorum (Safety Brake): By requiring a human start (ΔSTART) and acknowledgment, we make sure no autonomous action takes place without an operator deliberately initiating it. The presence gate is a hard stop – no Δ will fire without ΔSTART as configured in the policy. This ensures accountability and an opportunity for human judgment before anything happens. Additionally, for the most critical actions (like deploying legal measures or organization-wide changes), a quorum of approvals is mandated. In our stub, we require at least 2 signatures in truthlock/keys/signed/ (e.g., from 2 out of 3 key holders). This mimics multi-sig authorization: at least two people (or authorities) must consent before the system “turns the key” on a lethal action. Quorum is a common strategy in security to prevent a single rogue operator or compromised account from wreaking havoc. Together, presence + quorum enforce multi-party control. The code will simply refuse to run those actions if quorum isn’t met, logging an error instead.

One-Line Math Logging (Impulse Proof): The system even allows logging a one-line mathematical representation of an impulse for the records: e.g., ΔS(t) = S(t₁) − S(t₀);  δ_hit(t) → impulse: ∫ δ_hit dt = 1  ⇒ emit {TRO, §1983, WCI}. This is a symbolic way to say: the change in state S from time t₀ to t₁ is computed, and if a delta-hit event occurs, it integrates to 1 (meaning a single whole action), thus it emits the legal actions (TRO, 1983, WCI). It’s not necessary for functionality, but it’s a clever logging of the philosophy behind the mechanism – reinforcing that any strike is a unit impulse (area = 1) event.

Legal “Nuke” Basis – Fraud on the Court: The mention of FRCP 60(d)(3) is a nod to the legal basis for an extreme remedy when fraud on the court is detected. Under Federal Rule of Civil Procedure 60(d)(3), courts have the power to set aside a judgment for fraud on the court (there’s essentially no time limit on this). This is an inherent power of the court to address the most egregious misconduct (such as bribery of a judge, fabrication of evidence by officers of the court, etc.). In the context of TruthLock, if 

[![Discord Follow](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fdiscord.com%2Fapi%2Finvites%2Fautogpt%3Fwith_counts%3Dtrue&query=%24.approximate_member_count&label=total%20members&logo=discord&logoColor=white&color=7289da)](https://discord.gg/autogpt) &ensp;
[![Twitter Follow](https://img.shields.io/twitter/follow/Auto_GPT?style=social)](https://twitter.com/Auto_GPT) &ensp;

**AutoGPT** is a powerful platform that allows you to create, deploy, and manage continuous AI agents that automate complex workflows. 

## Hosting Options 
   - Download to self-host (Free!)
   - [Join the Waitlist](https://bit.ly/3ZDijAI) for the cloud-hosted beta (Closed Beta - Public release Coming Soon!)

## How to Self-Host the AutoGPT Platform
> [!NOTE]
> Setting up and hosting the AutoGPT Platform yourself is a technical process. 
> If you'd rather something that just works, we recommend [joining the waitlist](https://bit.ly/3ZDijAI) for the cloud-hosted beta.

### System Requirements

Before proceeding with the installation, ensure your system meets the following requirements:

#### Hardware Requirements
- CPU: 4+ cores recommended
- RAM: Minimum 8GB, 16GB recommended
- Storage: At least 10GB of free space

#### Software Requirements
- Operating Systems:
  - Linux (Ubuntu 20.04 or newer recommended)
  - macOS (10.15 or newer)
  - Windows 10/11 with WSL2
- Required Software (with minimum versions):
  - Docker Engine (20.10.0 or newer)
  - Docker Compose (2.0.0 or newer)
  - Git (2.30 or newer)
  - Node.js (16.x or newer)
  - npm (8.x or newer)
  - VSCode (1.60 or newer) or any modern code editor

#### Network Requirements
- Stable internet connection
- Access to required ports (will be configured in Docker)
- Ability to make outbound HTTPS connections

### Updated Setup Instructions:
We've moved to a fully maintained and regularly updated documentation site.

👉 [Follow the official self-hosting guide here](https://docs.agpt.co/platform/getting-started/)


This tutorial assumes you have Docker, VSCode, git and npm installed.

---

#### ⚡ Quick Setup with One-Line Script (Recommended for Local Hosting)

Skip the manual steps and get started in minutes using our automatic setup script.

For macOS/Linux:
```
curl -fsSL https://setup.agpt.co/install.sh -o install.sh && bash install.sh
```

For Windows (PowerShell):
```
powershell -c "iwr https://setup.agpt.co/install.bat -o install.bat; ./install.bat"
```

This will install dependencies, configure Docker, and launch your local instance — all in one go.

### 🧱 AutoGPT Frontend

The AutoGPT frontend is where users interact with our powerful AI automation platform. It offers multiple ways to engage with and leverage our AI agents. This is the interface where you'll bring your AI automation ideas to life:

   **Agent Builder:** For those who want to customize, our intuitive, low-code interface allows you to design and configure your own AI agents. 
   
   **Workflow Management:** Build, modify, and optimize your automation workflows with ease. You build your agent by connecting blocks, where each block     performs a single action.
   
   **Deployment Controls:** Manage the lifecycle of your agents, from testing to production.
   
   **Ready-to-Use Agents:** Don't want to build? Simply select from our library of pre-configured agents and put them to work immediately.
   
   **Agent Interaction:** Whether you've built your own or are using pre-configured agents, easily run and interact with them through our user-friendly      interface.

   **Monitoring and Analytics:** Keep track of your agents' performance and gain insights to continually improve your automation processes.

[Read this guide](https://docs.agpt.co/platform/new_blocks/) to learn how to build your own custom blocks.

### 💽 AutoGPT Server

The AutoGPT Server is the powerhouse of our platform This is where your agents run. Once deployed, agents can be triggered by external sources and can operate continuously. It contains all the essential components that make AutoGPT run smoothly.

   **Source Code:** The core logic that drives our agents and automation processes.
   
   **Infrastructure:** Robust systems that ensure reliable and scalable performance.
   
   **Marketplace:** A comprehensive marketplace where you can find and deploy a wide range of pre-built agents.

### 🐙 Example Agents

Here are two examples of what you can do with AutoGPT:

1. **Generate Viral Videos from Trending Topics**
   - This agent reads topics on Reddit.
   - It identifies trending topics.
   - It then automatically creates a short-form video based on the content. 

2. **Identify Top Quotes from Videos for Social Media**
   - This agent subscribes to your YouTube channel.
   - When you post a new video, it transcribes it.
   - It uses AI to identify the most impactful quotes to generate a summary.
   - Then, it writes a post to automatically publish to your social media. 

These examples show just a glimpse of what you can achieve with AutoGPT! You can create customized workflows to build agents for any use case.

---

### **License Overview:**

🛡️ **Polyform Shield License:**
All code and content within the `autogpt_platform` folder is licensed under the Polyform Shield License. This new project is our in-developlemt platform for building, deploying and managing agents.</br>_[Read more about this effort](https://agpt.co/blog/introducing-the-autogpt-platform)_

🦉 **MIT License:**
All other portions of the AutoGPT repository (i.e., everything outside the `autogpt_platform` folder) are licensed under the MIT License. This includes the original stand-alone AutoGPT Agent, along with projects such as [Forge](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/forge), [agbenchmark](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/benchmark) and the [AutoGPT Classic GUI](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/frontend).</br>We also publish additional work under the MIT Licence in other repositories, such as [GravitasML](https://github.com/Significant-Gravitas/gravitasml) which is developed for and used in the AutoGPT Platform. See also our MIT Licenced [Code Ability](https://github.com/Significant-Gravitas/AutoGPT-Code-Ability) project.

---
### Mission
Our mission is to provide the tools, so that you can focus on what matters:

- 🏗️ **Building** - Lay the foundation for something amazing.
- 🧪 **Testing** - Fine-tune your agent to perfection.
- 🤝 **Delegating** - Let AI work for you, and have your ideas come to life.

Be part of the revolution! **AutoGPT** is here to stay, at the forefront of AI innovation.

**📖 [Documentation](https://docs.agpt.co)**
&ensp;|&ensp;
**🚀 [Contributing](CONTRIBUTING.md)**

---
## 🤖 AutoGPT Classic
> Below is information about the classic version of AutoGPT.

**🛠️ [Build your own Agent - Quickstart](classic/FORGE-QUICKSTART.md)**

### 🏗️ Forge

**Forge your own agent!** &ndash; Forge is a ready-to-go toolkit to build your own agent application. It handles most of the boilerplate code, letting you channel all your creativity into the things that set *your* agent apart. All tutorials are located [here](https://medium.com/@aiedge/autogpt-forge-e3de53cc58ec). Components from [`forge`](/classic/forge/) can also be used individually to speed up development and reduce boilerplate in your agent project.

🚀 [**Getting Started with Forge**](https://github.com/Significant-Gravitas/AutoGPT/blob/master/classic/forge/tutorials/001_getting_started.md) &ndash;
This guide will walk you through the process of creating your own agent and using the benchmark and user interface.

📘 [Learn More](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/forge) about Forge

### 🎯 Benchmark

**Measure your agent's performance!** The `agbenchmark` can be used with any agent that supports the agent protocol, and the integration with the project's [CLI] makes it even easier to use with AutoGPT and forge-based agents. The benchmark offers a stringent testing environment. Our framework allows for autonomous, objective performance evaluations, ensuring your agents are primed for real-world action.

<!-- TODO: insert visual demonstrating the benchmark -->

📦 [`agbenchmark`](https://pypi.org/project/agbenchmark/) on Pypi
&ensp;|&ensp;
📘 [Learn More](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/benchmark) about the Benchmark

### 💻 UI

**Makes agents easy to use!** The `frontend` gives you a user-friendly interface to control and monitor your agents. It connects to agents through the [agent protocol](#-agent-protocol), ensuring compatibility with many agents from both inside and outside of our ecosystem.

<!-- TODO: insert screenshot of front end -->

The frontend works out-of-the-box with all agents in the repo. Just use the [CLI] to run your agent of choice!

📘 [Learn More](https://github.com/Significant-Gravitas/AutoGPT/tree/master/classic/frontend) about the Frontend

### ⌨️ CLI

[CLI]: #-cli

To make it as easy as possible to use all of the tools offered by the repository, a CLI is included at the root of the repo:

```shell
$ ./run
Usage: cli.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  agent      Commands to create, start and stop agents
  benchmark  Commands to start the benchmark and list tests and categories
  setup      Installs dependencies needed for your system.
```

Just clone the repo, install dependencies with `./run setup`, and you should be good to go!

## 🤔 Questions? Problems? Suggestions?

### Get help - [Discord 💬](https://discord.gg/autogpt)

[![Join us on Discord](https://invidget.switchblade.xyz/autogpt)](https://discord.gg/autogpt)

To report a bug or request a feature, create a [GitHub Issue](https://github.com/Significant-Gravitas/AutoGPT/issues/new/choose). Please ensure someone else hasn't created an issue for the same topic.

## 🤝 Sister projects

### 🔄 Agent Protocol

To maintain a uniform standard and ensure seamless compatibility with many current and future applications, AutoGPT employs the [agent protocol](https://agentprotocol.ai/) standard by the AI Engineer Foundation. This standardizes the communication pathways from your agent to the frontend and benchmark.

---

## Stars stats

<p align="center">
<a href="https://star-history.com/#Significant-Gravitas/AutoGPT">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=Significant-Gravitas/AutoGPT&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=Significant-Gravitas/AutoGPT&type=Date" />
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=Significant-Gravitas/AutoGPT&type=Date" />
  </picture>
</a>
</p>


## ⚡ Contributors

<a href="https://github.com/Significant-Gravitas/AutoGPT/graphs/contributors" alt="View Contributors">
  <img src="https://contrib.rocks/image?repo=Significant-Gravitas/AutoGPT&max=1000&columns=10" alt="Contributors" />
</a>
