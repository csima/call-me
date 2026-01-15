# SMS Support Design

Add SMS text messaging as an alternative to phone calls for Claude-to-user communication.

## Overview

Users can configure a default contact method (call or text) and override it per-request. SMS conversations are interactive: Claude texts, user replies, Claude processes the reply and continues.

## Configuration

**New environment variable:**

| Variable | Default | Description |
|----------|---------|-------------|
| `CALLME_DEFAULT_METHOD` | `call` | Default contact method: `call` or `text` |

Existing phone provider credentials (Twilio/Telnyx) already support SMS.

**Override in prompts:**

Users override naturally: "text me when done", "call me about this one".

**Tool schema:**

```typescript
call_user({
  message: string,           // What to say/text
  method?: "call" | "text"   // Optional, defaults to CALLME_DEFAULT_METHOD
})
```

## Conversation Flow

1. Claude calls `call_user({ message: "...", method: "text" })`
2. Plugin sends SMS via Twilio/Telnyx
3. Plugin blocks, waiting for incoming SMS webhook
4. User replies via text
5. Webhook receives reply, returns it to Claude
6. Claude decides: need more input? Call tool again. Otherwise, continue working.

Timeout after 30 minutes returns a timeout message so Claude can decide what to do.

## Provider Implementation

**Interface additions:**

```typescript
interface PhoneProvider {
  // Existing
  makeCall(to: string, webhookUrl: string): Promise<CallSession>

  // New
  sendSms(to: string, message: string): Promise<SmsSession>
  parseSmsWebhook(req: Request): SmsMessage | null
  getSmsAckResponse(): string
}
```

**Twilio:**

- Send: `client.messages.create({ to, from, body })`
- Receive: webhook POST to `/sms` with `From`, `To`, `Body` fields
- Requires separate SMS webhook URL config in Twilio console

**Telnyx:**

- Send: `client.messages.create({ to, from, text })`
- Receive: same webhook URL as voice, event type `message.received`

## Webhook Server

New `/sms` endpoint:

```typescript
app.post('/sms', async (req, res) => {
  const provider = getPhoneProvider()
  const message = provider.parseSmsWebhook(req)

  if (message && activeSmsSession?.from === message.from) {
    activeSmsSession.resolve(message.text)
  }

  res.status(200).send(provider.getSmsAckResponse())
})
```

Reuse existing webhook signature verification for security.

## Multi-Turn Conversations

The tool returns after each user reply. Claude controls whether to continue the conversation by calling the tool again with a follow-up question. This keeps conversation logic in Claude rather than the plugin.

## Files to Modify

| File | Changes |
|------|---------|
| `server/src/providers/types.ts` | Add SMS methods to interface |
| `server/src/providers/phone-twilio.ts` | Implement SMS methods |
| `server/src/providers/phone-telnyx.ts` | Implement SMS methods |
| `server/src/index.ts` | Add `/sms` webhook endpoint |
| `server/src/phone-call.ts` | Add SMS handling, support `method` param |
| `README.md` | Document new config and SMS setup |

## Edge Cases

- **User texts before Claude does**: Ignore (no active session)
- **User sends multiple texts**: Use first reply, ignore subsequent until next Claude message
- **Timeout**: Return conversation so far, let Claude decide to retry or continue
- **SMS not enabled on provider**: Clear error on first send attempt

## Out of Scope

- MMS/image support
- Group texts
- Read receipts
- Message threading
- Automatic SMS-to-call escalation
