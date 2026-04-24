---
name: steer-mcp
description: Use when calling the Steer MCP server for querying professional networks, managing relationships, viewing/editing user profile, or finding connection paths between people.
---

# Steer MCP SKILL Guide

Steer is a relationship-intelligence API exposed as an MCP server

## Contents

1. What this is
2. Connect
   2.1 Handshake
   2.2 Calling a tool
   2.3 Keeping this file up to date
3. Tools
   3.1 initialize
   3.2 user — usage, update
   3.3 ask_steer
   3.4 person — search, view, add, path
   3.5 relationship — search, view, add, edit, remove, group
   3.6 profile — view, edit
   3.7 follow — view, add, remove
4. Tool-chain cheat sheet


## 1. What this is

Seven tools across three groups:

- **System** — `initialize`, `user`
- **Relationship Intelligence** (read-only) — `ask_steer`, `person`
- **Relationship Management** (read & write) — `relationship`, `follow`, `profile`

Every request is a JSON-RPC 2.0 call to a single endpoint. A session ID is issued on `initialize` and required on every subsequent call.

---

## 2. Connect

> **No API key yet?** Sign up at https://app.steerai.ca and get your API key.

All traffic goes to one URL:

```
POST https://app.steerai.ca/api/mcp
```

Required headers:

```
Authorization: Bearer <API_KEY>
Content-Type: application/json
Mcp-Session-Id: <session-id>   # required after initialize
```

If the client cannot set custom headers, pass the key as a query parameter instead: `?key=<API_KEY>`.

> **Claude.ai web & desktop users:** don't follow this guide. The Steer MCP endpoint supports OAuth 2.1 for Claude's Custom Connector dialog — paste `https://app.steerai.ca/api/mcp` into Settings → Connectors → Add custom connector and authorize in the browser. Your Bearer key and any Claude OAuth connections share one credit pool; see the "Claude" tab on your Steer `/api` page to manage them. Agent and CLI clients (Claude Code, OpenCode, Cursor, etc.) should continue to use the Bearer flow described below.

### 2.1 Handshake

Call `initialize` first. Use `curl -i` so the response headers are visible — `Mcp-Session-Id` is returned there, not in the body.

```bash
curl -i -X POST https://app.steerai.ca/api/mcp \
  -H "Authorization: Bearer $STEER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "my-client", "version": "1.0" }
    }
  }'
```

Response headers include:

```
Mcp-Session-Id: 550e8400-e29b-41d4-a716-446655440000
```

Capture that value. Send it as the `Mcp-Session-Id` header on every subsequent request. If a later call returns a session error, call `initialize` again and retry.

### 2.2 Calling a tool

Tool calls use the `tools/call` method and may stream back as `text/event-stream`. Read `data:` lines until a JSON-RPC frame with a `result` arrives; that frame is the response.

```bash
curl -N -X POST https://app.steerai.ca/api/mcp \
  -H "Authorization: Bearer $STEER_API_KEY" \
  -H "Mcp-Session-Id: $STEER_SESSION_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "ask_steer",
      "arguments": { "question": "Who in my network works in climate tech?" }
    }
  }'
```

### 2.3 Keeping this file up to date

**On every `initialize` call**, check `result.serverInfo.version` in the response:

- If this file has no recorded version, or if the recorded version differs from `serverInfo.version`: tell the user the server is now on version `{serverInfo.version}` and that their SKILL.md is out of date, then call `{ "name": "user", "arguments": { "action": "update" } }` to fetch a fresh copy and replace this file before continuing.
- If the versions match: proceed normally.

```
Current version: 1.2.3
```

After replacing the file, re-read it before proceeding.

---

## 3. Tools

Every tool call uses this envelope:

```json
{
  "jsonrpc": "2.0",
  "id": <n>,
  "method": "tools/call",
  "params": { "name": "<tool>", "arguments": { ... } }
}
```

Tools that support multiple operations take an `action` field inside `arguments`.

---

### 3.1 `initialize`

Protocol handshake. Must be the first call. See §2.1.

Example response (note: `initialize` has no `structuredContent` — the whole `result` is the payload, and the session ID lives in the response headers):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": { "tools": {} },
    "serverInfo": { "name": "steer-mcp", "version": "1.0.0" }
  }
}
```

All subsequent tool calls return the envelope `{ jsonrpc, id, result: { content, structuredContent } }`. The examples below show only the `structuredContent` payload — that is the machine-readable body the caller should consume.

---

### 3.2 `user`

Account utility actions.

#### `action: "usage"`

Report remaining capacity in the current period and week windows. Does not modify anything.

```json
{ "name": "user", "arguments": { "action": "usage" } }
```

Example response (`structuredContent`):

```json
{
  "chat": {
    "period": { "remaining": 75, "reset_at": "2026-04-21T18:00:00.000Z" },
    "week":   { "remaining": 82, "reset_at": "2026-04-27T00:00:00.000Z" }
  },
  "app": {
    "period": { "remaining": 88, "reset_at": "2026-04-21T18:00:00.000Z" },
    "week":   { "remaining": 90, "follow_count": 3, "reset_at": "2026-04-27T00:00:00.000Z" }
  }
}
```

#### `action: "update"`

Fetch the latest version of this SKILL.md guide from the server and replace your local copy. Call this whenever `initialize` reports a version mismatch (see §0).

```json
{ "name": "user", "arguments": { "action": "update" } }
```

Example response (`structuredContent`):

```json
{ "ok": true, "content": "---\nname: steer-mcp\n..." }
```

Write the returned `content` string to your local SKILL.md file, then re-read it before continuing.

---

### 3.3 `ask_steer`

Natural-language entry point. Use this first for open-ended questions about people, connections, or network news. The parameter is `question`, **not** `message`.

```json
{
  "name": "ask_steer",
  "arguments": {
    "question": "How am I connected to Sarah Chen at Acme?"
  }
}
```

People referenced in the answer are annotated as `[[refid:Full Name]]` — strip the annotation when showing text to the user, and keep the `refid` for follow-up calls. Session memory persists within a session, so follow-up questions work naturally.

Example response (`structuredContent`):

```json
{
  "answer": "You're connected to [[p_abc123:Sarah Chen]] through [[p_def456:Mike Johnson]], who worked with Sarah at TechCorp in 2022.",
  "people": [
    { "refid": "p_abc123", "full_name": "Sarah Chen" },
    { "refid": "p_def456", "full_name": "Mike Johnson" }
  ],
  "entities": [
    { "type": "company",  "value": "Acme" },
    { "type": "company",  "value": "TechCorp" }
  ],
  "queries_executed": 4,
  "news_sources": [
    { "domain": "techcrunch.com", "name": "TechCrunch", "count": 2 }
  ],
  "evidence_counts": { "people_considered": 45, "text_matches": 8, "news_items": 2 }
}
```

---

### 3.4 `person`

Search for, view, add, or path-find a person in your network.

#### `action: "search"`

Full-text search across name, company, job, industry, location. Optional `limit` (default 9, max 20).

```json
{
  "name": "person",
  "arguments": {
    "action": "search",
    "query": "Sarah Chen",
    "limit": 9
  }
}
```

Example response (`structuredContent`):

```json
{
  "results": [
    {
      "refid": "p_xyz789",
      "full_name": "Sarah Chen",
      "job": "VP Engineering",
      "company": "Acme",
      "location": "Seattle, WA",
      "closeness": 0.825
    },
    {
      "refid": "p_uvw456",
      "full_name": "Sarah Chen",
      "job": null,
      "company": "TechCorp",
      "location": null,
      "closeness": 0.612
    }
  ]
}
```

#### `action: "view"`

Requires either `refid` or `relationship_id` (the latter resolves to the underlying person). Optional `sections`: subset of `["core", "news", "attributes", "network"]`. Omit for all.

```json
{
  "name": "person",
  "arguments": {
    "action": "view",
    "refid": "p_xyz789",
    "sections": ["core", "news"]
  }
}
```

Example response (`structuredContent`, all sections):

```json
{
  "refid": "p_xyz789",
  "full_name": "Sarah Chen",
  "job": "VP Engineering",
  "company": "Acme",
  "location": "Seattle, WA",
  "industry": "Technology",
  "is_hiring": true,
  "bio": "Building cloud infrastructure. Previously at TechCorp.",
  "estimated_goal": "Scaling engineering teams",
  "estimated_needs_wants": "Senior backend engineers",
  "hobbies": ["hiking", "photography"],
  "past_work_experience": ["Staff Engineer at TechCorp", "Senior Engineer at WebScale"],
  "linkedin": "https://linkedin.com/in/sarahchen",
  "twitter": "https://twitter.com/sarahchen",
  "instagram": null,
  "recent_news": [
    {
      "key_insight": "Sarah promoted to VP of Engineering",
      "topic": "Career",
      "date": "2026-03-15",
      "sentiment": "positive",
      "source_url": "https://example.com/news/1"
    }
  ],
  "connection": {
    "closeness": 0.825,
    "hops": 2,
    "path_names": ["You", "Mike Johnson", "Sarah Chen"]
  }
}
```

#### `action: "add"`

Request public data to be found for a person of your choice. Useful if they cannot be found through search. Requires at least one of `linkedin` (URL or bare handle), `refid`, or `relationship_id`. All other fields (`firstName`, `lastName`, `location`, `jobTitle`, `company`, `industry`) are optional seed data.

```json
{
  "name": "person",
  "arguments": {
    "action": "add",
    "linkedin": "sarahchen",
    "company": "Acme"
  }
}
```

Example response (`structuredContent`):

```json
{
  "ok": true,
  "handle": "sarahchen",
  "refid": "p_xyz789",
  "message": "Lookup started for sarahchen."
}
```

Process may take a few minutes. Check back with `person view` to see when data is available.

#### `action: "path"`

Find the warmest (shortest + strongest) connection path between two people. `to` is required. `from` is optional — omit to start from yourself.

```json
{
  "name": "person",
  "arguments": {
    "action": "path",
    "to": "abc123",
    "from": "def456"
  }
}
```

Example response (`structuredContent`):

```json
{
  "ok": true,
  "no_path": false,
  "nodes": [
    { "refid": "def456", "full_name": "You", "job": "Head of Growth" },
    { "refid": "ghi789", "full_name": "Mike Johnson", "job": "Partner at Acme Ventures" },
    { "refid": "abc123", "full_name": "Sarah Chen", "job": "VP Engineering" }
  ],
  "edges": [
    { "from": "def456", "to": "ghi789", "strength": 85 },
    { "from": "ghi789", "to": "abc123", "strength": 72 }
  ],
  "total_strength": 157,
  "hops": 2
}
```

If no path found, `no_path` is `true` and the result is empty.

---

### 3.5 `relationship`

Search, view, add, edit, or remove a relationship.

#### `action: "search"`

All filters are optional. With no filters, returns the 20 most recently modified relationships. Filters: `name`, `company`, `role`, `location`, `group` (partial match); `isFavorite` (boolean); `refid` (resolve a person to a relationship); `limit` (default 20, max 50).

```json
{
  "name": "relationship",
  "arguments": {
    "action": "search",
    "company": "Scotiabank",
    "isFavorite": true,
    "limit": 20
  }
}
```

Example response (`structuredContent`):

```json
{
  "results": [
    {
      "relationship_id": "rel_12345",
      "name": "James Wilson",
      "employer": "Scotiabank",
      "position": "Director of Sales",
      "custom_groups": ["Investors", "Key Contacts"],
      "is_favorite": true,
      "refid": "p_xyz789",
      "importance_context": null
    },
    {
      "relationship_id": "rel_67890",
      "name": "Lisa Park",
      "employer": "Scotiabank",
      "position": null,
      "custom_groups": [],
      "is_favorite": false,
      "refid": null,
      "importance_context": null
    }
  ]
}
```

#### `action: "view"`

Requires either `relationship_id` or `refid`. Optional `sections`: subset of `["core", "contact", "notes", "experience", "details"]`. Omit for all.

```json
{
  "name": "relationship",
  "arguments": {
    "action": "view",
    "relationship_id": "rel_12345",
    "sections": ["notes"]
  }
}
```

Example response (`structuredContent`, all sections — most fields may be `null` on a newly-added relationship):

```json
{
  "relationship_id": "rel_12345",
  "name": "James Wilson",
  "employer": "Scotiabank",
  "position": "Director of Sales",
  "location": "New York, NY",
  "avatar": "https://example.com/avatar.jpg",
  "is_favorite": true,
  "custom_groups": ["Investors", "Key Contacts"],
  "locked": false,
  "refid": "p_xyz789",
  "preferred_contact": "james@scotiabank.com",
  "date_created": "2025-01-15T10:30:00Z",
  "date_modified": "2026-04-10T14:22:00Z",
  "about": "Long-time client and trusted advisor",
  "industry": "Financial Services",
  "seniority": "C-Level",
  "contact_methods": [
    { "id": 1, "platform": "email", "type": "work", "value": "james@scotiabank.com", "is_preferred": true },
    { "id": 2, "platform": "phone", "type": "work", "value": "+1-555-0123", "is_preferred": false }
  ],
  "socials": [
    { "id": 1, "platform": "LinkedIn", "handle": "jameswilson", "url": "https://linkedin.com/in/jameswilson", "is_favorite": true }
  ],
  "notes": [
    { "id": 42, "title": "Follow-up", "content": "Discussed Q2 roadmap", "label": ["meeting"], "timestamp": "2026-04-01T09:15:00Z" }
  ],
  "interests": "Golf, venture capital",
  "known_from": "Client introduction in 2020",
  "reflections": "Strong negotiator, values long-term partnerships"
}
```

#### `action: "add"`

Requires `firstName` and `lastName`. Pass `linkedin` (URL or handle) or `refid`. Pass `noFollow: true` when bulk-adding to skip adding the person to your follow list.

```json
{
  "name": "relationship",
  "arguments": {
    "action": "add",
    "firstName": "Jane",
    "lastName": "Doe",
    "linkedin": "janed",
    "company": "Acme",
    "position": "Head of Partnerships",
    "customGroup": ["Investors"],
    "noFollow": true
  }
}
```

Example response (`structuredContent`):

```json
{
  "success": true,
  "relationship_id": "rel_54321",
  "name": "Jane Doe",
  "triggered": true,
  "following": false
}
```

- `triggered` — true if a public-data lookup was started
- `following` — true if the relationship was added to your follow list

#### `action: "edit"`

Requires `relationship_id`. Any core field from `add` can be set directly. `notes`, `contact_methods`, and `socials` take arrays of operation objects — each object carries its own `action` (`"add"`, `"update"`, `"delete"`, or `"set_preferred"`) and the target ID when updating or deleting an existing item.

```json
{
  "name": "relationship",
  "arguments": {
    "action": "edit",
    "relationship_id": "rel_12345",
    "position": "VP of Sales",
    "notes": [
      {
        "action": "add",
        "title": "YC event",
        "content": "Open to reconnecting.",
        "label": ["meeting"]
      }
    ],
    "contact_methods": [
      {
        "action": "add",
        "platform": "email",
        "type": "work",
        "value": "jane@acme.com",
        "is_preferred": true
      }
    ]
  }
}
```

Example response (`structuredContent`):

```json
{
  "success": true,
  "relationship_id": "rel_12345",
  "changes": {
    "card_fields": [],
    "content_fields": [],
    "notes":    { "added": 1, "updated": 0, "deleted": 0 },
    "contacts": { "added": 0, "deleted": 0 },
    "socials":  { "added": 0, "deleted": 0 }
  }
}
```

#### `action: "remove"`

Requires `relationship_id`. Permanently deletes the relationship.

```json
{
  "name": "relationship",
  "arguments": {
    "action": "remove",
    "relationship_id": "rel_12345"
  }
}
```

Example response (`structuredContent`):

```json
{ "removed": true, "relationship_id": "rel_12345" }
```

#### `action: "group"`

List all custom relationship-group tags the user has created. No other arguments.

```json
{
  "name": "relationship",
  "arguments": {
    "action": "group"
  }
}
```

Example response (`structuredContent`):

```json
{ "groups": ["Investors", "Sales Team", "Board Members", "Key Contacts"] }
```

---

### 3.6 `profile`

View or edit the authenticated user's own profile.

#### `action: "view"`

Optional `sections`: subset of `["core", "education", "experience", "skills", "certs", "accomplishments"]`. Omit for all.

```json
{
  "name": "profile",
  "arguments": {
    "action": "view",
    "sections": ["core", "experience"]
  }
}
```

Example response (`structuredContent`, all sections):

```json
{
  "core": {
    "nameFull": "John Example",
    "role": "Head of Growth",
    "company": "TechStartup Inc",
    "location": "Boston, MA",
    "linkedinUrl": "https://linkedin.com/in/johnexample",
    "networkingGoal": "Connect with engineering leaders",
    "lookingForJob": false,
    "preferredMedium": "Both",
    "industries": ["SaaS", "Technology"],
    "roles": ["Product Management", "Operations"],
    "locationPreferences": ["NYC", "SF", "Boston"],
    "companyPreferences": ["Series B+", "Growth Stage"]
  },
  "experience": [
    { "id": 1, "role": "VP Growth", "company": "PreviousCo", "description": null, "startDate": "2022-01", "endDate": null, "currentlyWorking": true }
  ],
  "education": [
    { "id": 1, "degree": "BS", "program": "Computer Science", "university": "MIT", "description": null, "startDate": null, "endDate": "2019", "graduated": true }
  ],
  "skills": [
    { "id": 1, "skill": "Product Management", "level": "Advanced" }
  ],
  "certs": [
    { "id": 1, "name": "Reforge Growth Strategy", "provider": "Reforge", "description": null, "date": "2023-06" }
  ],
  "accomplishments": [
    { "id": 1, "title": "Scaled user base 10x", "description": null }
  ],
  "groups": [
    { "id": 1, "name": "Boston Tech Leaders" }
  ]
}
```

#### `action: "edit"`

`sections`: subset of `["core", "education", "experience", "skills", "certs", "accomplishments"]`.

```json
{
  "name": "profile",
  "arguments": {
    "action": "edit",
    "experience": [{ "action": "add", "role": "CTO", "company": "Acme", "currentlyWorking": true }]
  }
}
```

Example response (`structuredContent`):

```json
{
  "success": true,
  "changes": {
    "core_fields":     [],
    "education":       { "added": 0, "updated": 0, "deleted": 0 },
    "experience":      { "added": 1, "updated": 0, "deleted": 0 },
    "skills":          { "added": 0, "deleted": 0 },
    "certs":           { "added": 0, "updated": 0, "deleted": 0 },
    "accomplishments": { "added": 0, "updated": 0, "deleted": 0 }
  }
}
```

---

### 3.7 `follow`

Manage the list of people you're following.

#### `action: "view"`

Returns the list of people you're currently following.

```json
{
  "name": "follow",
  "arguments": {
    "action": "view"
  }
}
```

Example response (`structuredContent`):

```json
{
  "action": "check",
  "following": [
    { "refid": "p_abc123", "person_name": "Sarah Chen" },
    { "refid": "p_def456", "person_name": "Mike Johnson" }
  ]
}
```

#### `action: "add"`

Requires either `refid` or `relationship_id`.

```json
{
  "name": "follow",
  "arguments": {
    "action": "add",
    "refid": "p_new789"
  }
}
```

Example response (`structuredContent`):

```json
{
  "action": "add",
  "following": [
    { "refid": "p_abc123", "person_name": "Sarah Chen" },
    { "refid": "p_new789", "person_name": "Alex Rodriguez" }
  ],
  "added": { "refid": "p_new789", "person_name": "Alex Rodriguez" }
}
```

#### `action: "remove"`

Requires either `refid` or `relationship_id`.

```json
{
  "name": "follow",
  "arguments": {
    "action": "remove",
    "relationship_id": "rel_12345"
  }
}
```

Example response (`structuredContent`):

```json
{
  "action": "remove",
  "following": [
    { "refid": "p_abc123", "person_name": "Sarah Chen" }
  ],
  "removed": { "refid": "p_def456", "person_name": "Mike Johnson" }
}
```

---

## 4. Tool-chain cheat sheet

Prefer the `ask_steer` tool for open-ended questions that may require multiple underlying queries, and use the more specific tools when you know exactly what you need. Here's a quick reference for common situations:

| Situation | Tool chain |
|---|---|
| "I have a meeting with X" | `person search` → `person view` → `relationship search` → `relationship view` |
| "Brief me on A, B, C" | `ask_steer` |
| "Who do I know at Company Y?" | `ask_steer` |
| "How am I connected to X?" | `person path` |
| "Find out more about someone" | `person add` with `linkedin`, `refid`, or `relationship_id` |
| "Who could introduce me to X?" | `ask_steer` → `relationship search` |
| "What's the latest news from my network?" | `ask_steer` |
| "Show my favourites" | `relationship search` with `isFavorite: true` |
| "Show my contacts at Scotiabank" | `relationship search` with `company: "Scotiabank"` |
| "Add a note to John's relationship" | `relationship search` → `relationship edit` |
| "Add this person as a contact" | `relationship add` with `refid` or `linkedin` |
| "Find public data about this person" | `person add` with `linkedin`, `refid`, or `relationship_id` |
| "What's on my profile?" | `profile view` |
| "Update my job title" | `profile edit` |
| "What groups do I have?" | `relationship group` |
| "Who am I following?" | `follow view` |
| "Follow David Kim" | `person search` → `follow add` |
