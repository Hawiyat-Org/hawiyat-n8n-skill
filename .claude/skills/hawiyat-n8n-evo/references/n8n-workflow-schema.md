# n8n Workflow JSON Schema Reference

## Top-Level Workflow Structure

```json
{
  "name": "Workflow Name",
  "active": false,
  "nodes": [...],
  "connections": {...},
  "settings": {
    "executionOrder": "v1"
  }
}
```

---

## Node Object Structure

```json
{
  "id": "unique-uuid",
  "name": "Node Display Name",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [250, 300],
  "parameters": {
    // node-specific parameters
  }
}
```

---

## Common Node Types

### Webhook Trigger
```json
{
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "parameters": {
    "path": "my-webhook-path",
    "httpMethod": "POST",
    "responseMode": "responseNode",
    "options": {}
  }
}
```

### Respond to Webhook
```json
{
  "type": "n8n-nodes-base.respondToWebhook",
  "typeVersion": 1,
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ { \"status\": \"ok\" } }}"
  }
}
```

### Code Node (JavaScript)
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "jsCode": "// your JavaScript code\nconst body = $input.first().json;\nreturn [{ json: { processed: body } }];"
  }
}
```

### HTTP Request Node
```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "={{ `https://${$vars.EVO_HOST}/message/sendText/${$vars.EVO_INSTANCE}` }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "apikey", "value": "={{ $vars.EVO_API_KEY }}" }
      ]
    },
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify({ number: $json.phone, text: $json.reply, delay: 1200 }) }}"
  }
}
```
> ⚠️ For typeVersion 4.2, always use `specifyBody: "json"` + `jsonBody` — never the old `"body"` or `"bodyParameters"` keys.

### IF Node
```json
{
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "parameters": {
    "conditions": {
      "options": { "caseSensitive": true },
      "conditions": [
        {
          "id": "cond-1",
          "leftValue": "={{ $json.data.key.fromMe }}",
          "rightValue": true,
          "operator": { "type": "boolean", "operation": "equals" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

### Schedule Trigger
```json
{
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "parameters": {
    "rule": {
      "interval": [{ "field": "hours", "hoursInterval": 1 }]
    }
  }
}
```

### Set Node (assign variables)
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "parameters": {
    "assignments": {
      "assignments": [
        { "id": "a1", "name": "userId", "value": "={{ $json.data.key.remoteJid }}", "type": "string" }
      ]
    }
  }
}
```

### Stop and Error Node
```json
{
  "type": "n8n-nodes-base.stopAndError",
  "typeVersion": 1,
  "parameters": {
    "errorMessage": "Execution stopped: loop guard triggered"
  }
}
```

---

## Connections Format

Connections map: `SourceNodeName.outputIndex → [{ node, type, index }]`

```json
"connections": {
  "Webhook Trigger": {
    "main": [
      [
        { "node": "Guard Filter", "type": "main", "index": 0 }
      ]
    ]
  },
  "Guard Filter": {
    "main": [
      [{ "node": "Fetch Memory", "type": "main", "index": 0 }],
      []
    ]
  }
}
```

- `main[0]` = first output (true/pass branch for IF nodes)
- `main[1]` = second output (false/fail branch for IF nodes)
- Empty array `[]` = dead end (stop execution)

---

## Static Data (Session Memory in n8n)

Use in Code node to persist data across executions:

```javascript
// Get static data
const staticData = $getWorkflowStaticData('global');
const sessionKey = `session_${userId}`;
const session = staticData[sessionKey] || { history: [], lastSeen: null };

// Add new turn
session.history.push({ role: 'user', content: userMessage });
session.history.push({ role: 'assistant', content: aiReply });

// Prune to last 10 turns
if (session.history.length > 20) {
  session.history = session.history.slice(-20);
}

// Save back
session.lastSeen = Date.now();
staticData[sessionKey] = session;
```

---

## n8n Expressions Cheatsheet

| Expression | Meaning |
|-----------|---------|
| `$json` | Current node's output JSON |
| `$input.first().json` | First item from input |
| `$node["NodeName"].json` | Output of a specific node |
| `$vars.VARIABLE_NAME` | n8n environment variable |
| `$workflow.id` | Current workflow ID |
| `$execution.id` | Current execution ID |

---

## Complete Chatbot Workflow Template (Production-Ready)

This is the **canonical 5-node chatbot** to use as the base for all WhatsApp AI chatbot requests.
It includes: guard, memory, AI call (OpenAI HTTP), typing indicator, and reply.

Flow: `Webhook → Guard+Memory → AI Call → Memory Update+Build Reply → Typing → Send Text`

```json
{
  "name": "Hawiyat WhatsApp AI Chatbot",
  "active": false,
  "nodes": [
    {
      "id": "a1b2c3d4-0001-4000-8000-000000000001",
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [100, 300],
      "parameters": {
        "path": "whatsapp-bot",
        "httpMethod": "POST",
        "responseMode": "lastNode"
      }
    },
    {
      "id": "a1b2c3d4-0002-4000-8000-000000000002",
      "name": "Guard + Fetch Memory",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [380, 300],
      "parameters": {
        "jsCode": "// === GUARD ===\nconst body = $input.first().json.body || $input.first().json;\nif (body?.data?.key?.fromMe === true) return [];\n\nconst remoteJid = body?.data?.key?.remoteJid;\nconst messageId = body?.data?.key?.id;\nconst userMessage = (\n  body?.data?.message?.conversation ||\n  body?.data?.message?.extendedTextMessage?.text || ''\n).trim();\n\nif (!userMessage || !remoteJid) return [];\n\n// === DEDUP ===\nconst staticData = $getWorkflowStaticData('global');\nconst dedupKey = `seen_${messageId}`;\nif (staticData[dedupKey]) return [];\nstaticData[dedupKey] = true;\n\n// Prune old dedup keys\nconst dedupKeys = Object.keys(staticData).filter(k => k.startsWith('seen_'));\nif (dedupKeys.length > 500) {\n  dedupKeys.slice(0, dedupKeys.length - 500).forEach(k => delete staticData[k]);\n}\n\n// === MEMORY ===\nconst sessionKey = `session_${remoteJid}`;\nconst session = staticData[sessionKey] || { history: [], lastSeen: 0 };\n\nif (Date.now() - session.lastSeen > 86400000) session.history = [];\n\n// Build messages array for AI\nconst messages = [\n  { role: 'system', content: 'You are a helpful assistant.' },\n  ...session.history.slice(-10),\n  { role: 'user', content: userMessage }\n];\n\n// Save user turn to session (AI reply added after)\nsession.history.push({ role: 'user', content: userMessage });\nsession.lastSeen = Date.now();\nstaticData[sessionKey] = session;\n\nreturn [{ json: { phone: remoteJid, sessionKey, messages, userMessage } }];"
      }
    },
    {
      "id": "a1b2c3d4-0003-4000-8000-000000000003",
      "name": "Call OpenAI",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [660, 300],
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Authorization", "value": "=Bearer {{ $vars.OPENAI_API_KEY }}" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ model: 'gpt-4o-mini', messages: $json.messages, max_tokens: 500, temperature: 0.7 }) }}",
        "options": { "response": { "response": { "fullResponse": false } } }
      }
    },
    {
      "id": "a1b2c3d4-0004-4000-8000-000000000004",
      "name": "Update Memory + Build Reply",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [940, 300],
      "parameters": {
        "jsCode": "const staticData = $getWorkflowStaticData('global');\nconst prev = $('Guard + Fetch Memory').first().json;\nconst sessionKey = prev.sessionKey;\nconst phone = prev.phone;\n\n// Extract AI reply from OpenAI response\nconst aiReply = $json?.choices?.[0]?.message?.content?.trim() || 'Sorry, I could not respond right now.';\n\n// Append assistant turn to session\nconst session = staticData[sessionKey];\nif (session) {\n  session.history.push({ role: 'assistant', content: aiReply });\n  staticData[sessionKey] = session;\n}\n\nreturn [{ json: { phone, reply: aiReply } }];"
      }
    },
    {
      "id": "a1b2c3d4-0005-4000-8000-000000000005",
      "name": "Send Typing Indicator",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1220, 300],
      "parameters": {
        "method": "POST",
        "url": "={{ `https://${$vars.EVO_HOST}/chat/sendPresence/${$vars.EVO_INSTANCE}` }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [{ "name": "apikey", "value": "={{ $vars.EVO_API_KEY }}" }]
        },
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ number: $json.phone, options: { presence: 'composing', delay: 2000 } }) }}"
      }
    },
    {
      "id": "a1b2c3d4-0006-4000-8000-000000000006",
      "name": "Send Reply",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1500, 300],
      "parameters": {
        "method": "POST",
        "url": "={{ `https://${$vars.EVO_HOST}/message/sendText/${$vars.EVO_INSTANCE}` }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [{ "name": "apikey", "value": "={{ $vars.EVO_API_KEY }}" }]
        },
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ number: $('Update Memory + Build Reply').first().json.phone, text: $('Update Memory + Build Reply').first().json.reply, delay: 1200 }) }}"
      }
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [[{ "node": "Guard + Fetch Memory", "type": "main", "index": 0 }]]
    },
    "Guard + Fetch Memory": {
      "main": [[{ "node": "Call OpenAI", "type": "main", "index": 0 }]]
    },
    "Call OpenAI": {
      "main": [[{ "node": "Update Memory + Build Reply", "type": "main", "index": 0 }]]
    },
    "Update Memory + Build Reply": {
      "main": [[{ "node": "Send Typing Indicator", "type": "main", "index": 0 }]]
    },
    "Send Typing Indicator": {
      "main": [[{ "node": "Send Reply", "type": "main", "index": 0 }]]
    }
  },
  "settings": { "executionOrder": "v1" }
}
```

### Required n8n environment variables (`$vars`):
| Variable | Value |
|----------|-------|
| `EVO_HOST` | e.g. `evo.hawiyat.cloud` |
| `EVO_INSTANCE` | e.g. `my-instance` |
| `EVO_API_KEY` | Evolution API key |
| `OPENAI_API_KEY` | OpenAI secret key |

### To use a different AI provider, replace the `Call OpenAI` node URL and body:
- **Anthropic Claude**: `https://api.anthropic.com/v1/messages` — change body to `{ model, messages, max_tokens }` + add `x-api-key` and `anthropic-version` headers
- **Local/Ollama**: `http://localhost:11434/api/chat` — adjust body to Ollama format
- **Any OpenAI-compatible API**: just swap the URL
