# Design QA Review — a Claude skill

Compares a Figma design against a staging build and files the differences as ClickUp tickets.

Give Claude a Figma URL and a staging URL. It maps every user flow and state, walks staging in your own Chrome, measures computed CSS from the live DOM (nothing eyeballed), annotates what's wrong on screenshots, and files ClickUp tickets. UX suggestions are kept separate from bugs.

Every review covers six checks: **Content · Component · Colour · Text · Spacing · Height & width**.

## Requirements

- [Claude in Chrome](https://claude.com/chrome) extension, signed in to your staging environment (primary engine for navigation, screenshots and annotation)
- Figma connector (used read-only)
- ClickUp connector (for tickets)

## Install

**Claude app (claude.ai / desktop / Cowork)**
1. Download this repo as a ZIP, or zip the `design-qa-review` folder.
2. Go to **Customize → Skills → Upload a skill** and choose the zip.

**Claude Code**
```bash
git clone https://github.com/shankarm-hiver/DesignQA.git
cp -r DesignQA/design-qa-review ~/.claude/skills/
```

## Use

```
Review <Figma URL> against <staging URL>
```

Once the findings are in, say "create all tickets", "critical and major only" or "skip minor ones".
