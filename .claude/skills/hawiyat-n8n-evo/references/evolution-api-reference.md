# Evolution API v2 — Endpoint Reference

Base URL: `https://<instance>.hawiyat.cloud`  
Auth header: `apikey: <API_KEY>`

---

## Instance Management

### Fetch All Instances
```http
GET /instance/fetchInstances
apikey: <GLOBAL_API_KEY>
```

### Connection State
```http
GET /instance/connectionState/{instanceName}
apikey: <API_KEY>
```
Response: `{ "instance": { "instanceName": "...", "state": "open" } }`

States: `open` | `connecting` | `close`

### Restart Instance
```http
PUT /instance/restart/{instanceName}
apikey: <API_KEY>
```

---

## Webhook Management

### Set Instance Webhook
```http
POST /webhook/set/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "url": "https://<n8n>.hawiyat.cloud/webhook/<path>",
  "webhook_by_events": false,
  "webhook_base64": false,
  "events": [
    "MESSAGES_UPSERT",
    "CONNECTION_UPDATE"
  ]
}
```

### Find Instance Webhook
```http
GET /webhook/find/{instanceName}
apikey: <API_KEY>
```

---

## Message Sending

### Send Text Message
```http
POST /message/sendText/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "number": "213XXXXXXXXX@s.whatsapp.net",
  "text": "Hello, this is your reply.",
  "delay": 1200
}
```
- `number` format: international number + `@s.whatsapp.net`
- `delay` in ms (simulates typing delay)

### Send Media (Image/Document/Video)
```http
POST /message/sendMedia/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "number": "213XXXXXXXXX@s.whatsapp.net",
  "mediatype": "image",
  "mimetype": "image/jpeg",
  "caption": "Optional caption",
  "media": "https://url-to-media.com/image.jpg",
  "fileName": "image.jpg"
}
```
- `mediatype`: `image` | `document` | `video` | `audio`

### Send Buttons (Interactive)
```http
POST /message/sendButtons/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "number": "213XXXXXXXXX@s.whatsapp.net",
  "title": "Choose an option",
  "description": "Pick one below",
  "footer": "Powered by Hawiyat",
  "buttons": [
    { "type": "reply", "displayText": "Option 1", "id": "opt1" },
    { "type": "reply", "displayText": "Option 2", "id": "opt2" }
  ]
}
```

---

## Chat Management

### Send Typing Presence (before replying)
```http
POST /chat/sendPresence/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "number": "213XXXXXXXXX@s.whatsapp.net",
  "options": {
    "presence": "composing",
    "delay": 2000
  }
}
```
Always call this before sending a reply to simulate human typing.

### Mark Messages as Read
```http
POST /chat/markMessageAsRead/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "readMessages": [
    {
      "id": "<messageId>",
      "fromMe": false,
      "remoteJid": "213XXXXXXXXX@s.whatsapp.net"
    }
  ]
}
```

### Check if Number is on WhatsApp
```http
POST /chat/whatsappNumbers/{instanceName}
apikey: <API_KEY>
Content-Type: application/json

{
  "numbers": ["213XXXXXXXXX"]
}
```

---

## Incoming Webhook Payload Structure

When `MESSAGES_UPSERT` fires, the payload is:

```json
{
  "event": "messages.upsert",
  "instance": "instanceName",
  "data": {
    "key": {
      "remoteJid": "213XXXXXXXXX@s.whatsapp.net",
      "fromMe": false,
      "id": "MESSAGE_ID_STRING"
    },
    "message": {
      "conversation": "User's text message here"
    },
    "messageType": "conversation",
    "messageTimestamp": 1718000000,
    "pushName": "Contact Name"
  }
}
```

### Extracting message content in n8n Code node:
```javascript
const body = $input.first().json.body || $input.first().json;

// Anti-loop guard
if (body?.data?.key?.fromMe === true) return [];

// Extract fields
const remoteJid = body?.data?.key?.remoteJid;
const messageId = body?.data?.key?.id;
const userMessage = body?.data?.message?.conversation 
  || body?.data?.message?.extendedTextMessage?.text 
  || '';
const senderName = body?.data?.pushName || 'User';

// Deduplication (use static data)
const staticData = $getWorkflowStaticData('global');
if (staticData[`seen_${messageId}`]) return []; // already processed
staticData[`seen_${messageId}`] = true;

// Only process text messages
if (!userMessage) return [];
```

---

## Full Chatbot Code Node Template

The complete Guard + Memory + AI prep logic belongs in a single Code node.
Use the **Complete Chatbot Workflow Template** in `n8n-workflow-schema.md` for the full importable JSON.

The Code node (Guard + Fetch Memory) handles:
1. fromMe guard
2. messageId deduplication
3. Message text extraction (text + extendedText)
4. Session memory fetch + 24h expiry
5. messages[] array construction for AI
6. User turn saved to session

The second Code node (Update Memory + Build Reply) handles:
1. Extracting AI reply from OpenAI response (`choices[0].message.content`)
2. Appending assistant turn to session memory
3. Outputting `{ phone, reply }` for the HTTP nodes

---

## Error Handling Pattern

Always add error output handling on HTTP Request nodes calling Evolution API:

```json
{
  "onError": "continueErrorOutput"
}
```

In the error branch: log the error and return a fallback message to the user.

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Not filtering `fromMe: true` | Always add guard at top of Code node |
| Sending to raw phone number | Always append `@s.whatsapp.net` |
| Storing full webhook body in memory | Extract only needed fields |
| No deduplication | Use `seen_<messageId>` in staticData |
| AI prompt with full history | Slice to last 10 messages only |
| Sending before "typing" indicator | Always call `sendPresence` first |
