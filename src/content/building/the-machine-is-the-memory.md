---
title: "The Machine Is the Memory"
date: 2026-03-28
description: "On building personal AI infrastructure that survives crashes, context loss, and your own forgetfulness."
draft: true
---

The conversation window is ephemeral. My AI assistant's context resets every time I close the tab. Everything it learned about me, my projects, my preferences, my codebase — gone. The next instance starts fresh, asking the same questions, making the same wrong assumptions.

I decided this was unacceptable. So I built a system where the machine itself is the memory.

---

## The Problem

I run three Mac Studios. One is my daily workstation (Hearth, M2 Max, 64GB). One is a compute workhorse (Anvil, M3 Ultra, 96GB). One is a mobile powerhouse (Ember, M5 Max, 128GB). Between them, I run AI agents — Claude Code instances — that do research, write code, manage my Obsidian vault, monitor my messaging streams, and orchestrate each other.

The problem: these agents crash. The machine reboots. The conversation window compresses. iCloud hiccups. Power goes out. And when any of these things happen, the agent forgets everything.

The traditional solution is "just start a new conversation." The real solution is: make the filesystem do the remembering.

---

## The Architecture

Every interaction produces durable artifacts. Nothing lives only in the conversation. The system has five layers:

| Layer | What Goes Here | Survives |
|-------|---------------|----------|
| **Session logs** | Timeline of what each agent did — files modified, decisions made, errors hit | Crashes, reboots, context resets |
| **Agent protocol** | Who's active, what they own, handoff notes between agents | Agent rotation, multi-day work |
| **Prompt store** | Every prompt I've sent, indexed with FTS5 full-text search | Total context loss — I can search my own history |
| **Vault** | Knowledge, projects, people, daily notes | Everything — this is the canonical store |
| **State files** | Per-agent state (JSON), task queues, dispatch logs | Agent crashes, session boundaries |

When a new agent boots, it reads all five layers. Within 30 seconds, it knows what happened yesterday, what's pending, who else is active, and what files it shouldn't touch. No human has to brief it. The machine briefs itself.

---

## Session Logging: The Constitutional Requirement

The most important design decision: session logs are written *continuously*, not at shutdown.

This seems obvious. It isn't. Every AI session logging system I've seen writes the log when the conversation ends. But conversations don't end gracefully — they crash, they timeout, they get killed by an errant `kill -9`, or the power goes out because you live in Oakland.

My session logs are append-only, auto-saving documents. After every significant action — file created, commit pushed, decision made — the agent appends to its log and rebuilds the index. The log is always current. If the machine crashes mid-sentence, the log already contains everything up to the previous action.

This is crash resilience for AI. Not checkpointing model weights — checkpointing *work*.

---

## Multi-Agent Coordination Without a Server

I don't run a coordination server. I don't use a message queue. I use the filesystem.

Each agent has a directory (`~/agents/<name>/`). Each agent has a CLAUDE.md that defines its mission, boundaries, and what files it owns. A shared `agent-protocol.md` file tracks who's active, what they changed, and what's next. A dispatch queue (`~/agents/dispatch/inbox/`) lets agents assign work to each other.

This is deliberately low-tech. The filesystem is the most reliable, debuggable, and inspectable coordination mechanism available. You can `cat` it. You can `grep` it. You can `git blame` it. You can read it on your phone over SSH. It doesn't go down when Redis goes down.

The agents coordinate through files the way scientists coordinate through papers: write your findings, publish them to a shared location, read what others published before you start your own work. It's asynchronous, durable, and embarrassingly simple.

---

## The Prompt Store

Every prompt I send to any AI agent gets stored in a SQLite database with FTS5 full-text search. This is the "undo" for context loss.

When I open a new conversation, the agent can search the prompt store: "What was I working on yesterday?" "When did I last ask about ImmuneBridge?" "What prompts mention the vault restructuring?" The answers are there, indexed, searchable, with timestamps and session IDs.

The prompt store contains 2,300+ prompts spanning dozens of sessions. It's my side of every AI conversation I've ever had. It's the most complete record of my technical decision-making that exists.

---

## What This Actually Feels Like

It feels like having a team that never forgets.

I can say "park" to an agent, close the tab, sleep, and open a new instance tomorrow. The new instance reads the session log, the agent protocol, the tasks file, and picks up exactly where the last one stopped. It knows the decisions that were made. It knows the files that are in flight. It knows the blockers.

This isn't memory in the human sense. It's memory in the infrastructure sense — like how a database remembers your transactions, or how git remembers your commits. The machine doesn't "recall" anything. It reads state from disk. But the effect is the same: continuity across interruptions.

The cost is discipline. Every session log has to be written. Every handoff has to be documented. Every decision has to be externalized. If it's not on disk, it doesn't exist. This is the founding principle: **the conversation window is ephemeral; the filesystem is the real memory.**

---

## The Practical Upshot

I've been running this system for two months. In that time, 45+ agents have operated across my machines. They've created session logs, handed off to each other, and maintained continuity across reboots, power outages, and my own forgetfulness.

The longest chain of continuity is 8 agents deep — eight consecutive instances, each reading the previous one's session log, each adding to the shared record. No human intervention required at the transitions.

The total cost: zero, beyond the Claude API subscription I was already paying. The entire system runs on SQLite, markdown files, Node.js scripts, and the filesystem. No cloud services. No external dependencies. No vendor lock-in.

If Anthropic disappeared tomorrow, I'd still have every session log, every prompt, every handoff, every decision — because it's all in files on my machine.

---

## What I'd Do Differently

The session index should have been there from day one. I built it on day 14 and had to retroactively index two weeks of session logs. Build the index first.

The prompt store should auto-capture, not rely on manual rebuilds. I'm working on a daemon that watches for new JSONL files and indexes in real-time.

The agent protocol needs better conflict resolution. When two agents try to edit the same file, the current system is "check ownership first" — but mistakes happen. A lock-file mechanism or git-style merge would be better.

And the whole thing needs to be open-sourced. That's next.

---

*Peretz Partensky runs three Mac Studios, 45+ AI agents, and an Obsidian vault that ties them all together. He lives in Oakland and records voice memos about the experience.*
