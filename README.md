# Agent Skills

Tailored workflow.

## AI Agent

Use [pi](https://pi.dev/). Its a surgical knife, precision at the expense of convenience.

NOTE: You will have to compact and manually add in web-browsing skills yourself. (Claude Code does that for you automatically).


```bash
npm install -g @mariozechner/pi-coding-agent
```

## Skills

You will need node.js for the majority of additional features.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
nvm install --lts

# User level
git clone https://github.com/badlogic/pi-skills ~/.pi/agent/skills/pi-skills

# Or project-level
git clone https://github.com/badlogic/pi-skills .pi/skills/pi-skills
```
