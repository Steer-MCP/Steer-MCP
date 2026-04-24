# Steer MCP

Steer is a relationship-intelligence platform. This MCP server gives your AI agent access to your professional network. Search people, manage contacts, find connection paths, and get AI-powered network insights.

**Sign up at [app.steerai.ca](https://app.steerai.ca)** to get your API key.

---

## Claude.ai web & desktop

Paste `https://app.steerai.ca/api/mcp` into **Settings → Connectors → Add custom connector** and authorize in the browser. No key or config file needed.

---

## OpenClaw, Claude Code, Cursor, OpenCode, and other CLI agents

Paste this prompt into your agent to get set up:

```
Download https://raw.githubusercontent.com/Steer-MCP/Steer-MCP/main/skills/steer-mcp/SKILL.md and save it to your project as SKILL.md. Then read it and follow the instructions inside.
```

The SKILL.md file contains everything your agent needs: the endpoint, authentication, and full tool reference.

---

## Endpoint

```
POST https://app.steerai.ca/api/mcp
```

```
Authorization: Bearer <YOUR_API_KEY>
```
