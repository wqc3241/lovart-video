# /lovart-video — Claude Code Skill

A reusable Claude Code command-skill that generates a video from a text
prompt (and optional reference images) via the Lovart AI agent
(`lovartai/lovart-skill`).

It is a low-level building block: it knows nothing about products or social
posts, so any skill that needs an AI-generated video can call it.

## Requirements

- `python3` and `curl`.
- `LOVART_ACCESS_KEY` and `LOVART_SECRET_KEY` set in `~/.openclaw/.env`.
- Network access (the skill bootstraps Lovart's `agent_skill.py` on first run).

## Install

Symlink the skill into your Claude Code commands directory:

```bash
ln -sf "$(pwd)/lovart-video.md" ~/.claude/commands/lovart-video.md
```

## Usage

```
/lovart-video --prompt "8-second vertical 9:16 product clip of ..." --ref ./photo.jpg --out ./out.mp4
```

See `lovart-video.md` for the full argument reference.
