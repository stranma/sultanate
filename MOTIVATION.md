# Sultanate -- Motivation

## The Problem

You want to run AI agents as your personal workforce. A coding agent on one
repo, a research agent on another, an assistant handling email. Each needs
internet access, credentials, and a workspace. You want to tell another
agent "spin this up" and have it working in seconds -- not write Docker
configs, network rules, and secret distribution for each one.

Today, every new agent is a project:

1. **Set up a container.** Install the runtime, configure networking, mount
   the workspace, set up proxy rules.
2. **Distribute secrets.** Create tokens, figure out which agent needs which
   credential, inject them safely. Every agent runtime does this differently.
3. **Worry about leaks.** The agent has your GitHub token. A confused LLM
   could POST your source code somewhere. You have no visibility or control
   over outbound traffic.
4. **Repeat for the next agent.** Different runtime? Start over. Want to try
   a new coding agent framework? Redo the whole setup.

This doesn't scale. You spend more time on infrastructure than on the work
the agents are supposed to do.

## What We Want

**Deploy fast.** Tell Vizier "create a coding agent for repo X" via Telegram
and have a working, isolated agent in seconds. No YAML, no manual Docker
setup, no credential wrangling. The deployment agent handles all of it.

**Stay reasonably safe.** Agents never hold dangerous secrets -- credentials
are injected at the network layer, invisible to the agent. Outbound writes
to non-whitelisted domains are blocked. If an agent needs something, it
asks. Not paranoid, but good enough that a confused agent can't leak your
code or tokens.

**Add new tech easily.** Want to try a new agent framework? Write a firman
(container template) and a berat (agent profile). The security layer, the
deployment, the credential injection -- all of that stays the same. The
platform doesn't care what runs inside the container as long as it speaks
HTTP through the proxy.

## How It Works

Sultanate separates three concerns:

- **What runs** (firman + berat) -- container template and agent profile.
  Swappable. Add a new runtime by writing a new firman. Add a new agent
  personality by writing a new berat.
- **How it's deployed** (Vizier) -- an agent that manages other agents.
  You talk to it, it creates containers, bootstraps workspaces, starts
  runtimes. CLI for scripting, Telegram for conversation.
- **How it's secured** (Janissary + Sentinel) -- a proxy that controls
  what goes in and out, and a trusted agent that manages secrets. The
  security layer is runtime-agnostic: it works the same whether the
  province runs Hermes, OpenHands, CrewAI, or anything else.
