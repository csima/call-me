# SMS Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add SMS text messaging as an alternative contact method to phone calls.

**Architecture:** Extend existing phone providers (Twilio/Telnyx) with SMS methods. Add `/sms` webhook endpoint to receive replies. Support `method` parameter on `initiate_call` tool with global default via environment variable.

**Tech Stack:** TypeScript, Twilio/Telnyx REST APIs, HTTP webhooks

---

### Task 1: Add SMS Types to Provider Interface

**Files:**
- Modify: `server/src/providers/types.ts`

**Step 1: Add SMS types and methods to PhoneProvider interface**

Add after line 39 (after `getStreamConnectXml`):

```typescript
  /**
   * Send an SMS message
   * @returns Message SID/ID from the provider
   */
  sendSms(to: string, from: string, message: string): Promise<string>;

  /**
   * Parse incoming SMS webhook request
   * @returns Parsed message or null if not an SMS webhook
   */
  parseSmsWebhook(body: string, contentType: string): SmsMessage | null;

  /**
   * Get acknowledgment response for SMS webhook
   */
  getSmsAckResponse(): { contentType: string; body: string };
```

Add after `PhoneConfig` interface (line 45):

```typescript
export interface SmsMessage {
  from: string;
  to: string;
  body: string;
  messageId: string;
}
```

**Step 2: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: Type errors in phone-twilio.ts and phone-telnyx.ts (missing SMS methods)

**Step 3: Commit**

```bash
git add server/src/providers/types.ts
git commit -m "feat(types): add SMS methods to PhoneProvider interface"
```

---

### Task 2: Implement Twilio SMS Methods

**Files:**
- Modify: `server/src/providers/phone-twilio.ts`

**Step 1: Add sendSms method**

Add after `getStreamConnectXml` method (after line 124):

```typescript
  async sendSms(to: string, from: string, message: string): Promise<string> {
    if (!this.accountSid || !this.authToken) {
      throw new Error('Twilio not initialized');
    }

    const auth = Buffer.from(`${this.accountSid}:${this.authToken}`).toString('base64');

    const response = await fetch(
      `https://api.twilio.com/2010-04-01/Accounts/${this.accountSid}/Messages.json`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Basic ${auth}`,
          'Content-Type': 'application/x-www-form-urlencoded',
        },
        body: new URLSearchParams({
          To: to,
          From: from,
          Body: message,
        }).toString(),
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Twilio SMS failed: ${response.status} ${error}`);
    }

    const data = await response.json() as { sid: string };
    return data.sid;
  }
```

**Step 2: Add parseSmsWebhook method**

Add after `sendSms`:

```typescript
  parseSmsWebhook(body: string, contentType: string): SmsMessage | null {
    if (!contentType.includes('application/x-www-form-urlencoded')) {
      return null;
    }

    const params = new URLSearchParams(body);
    const from = params.get('From');
    const to = params.get('To');
    const messageBody = params.get('Body');
    const messageSid = params.get('MessageSid');

    if (!from || !to || !messageBody || !messageSid) {
      return null;
    }

    return {
      from,
      to,
      body: messageBody,
      messageId: messageSid,
    };
  }
```

**Step 3: Add getSmsAckResponse method**

Add after `parseSmsWebhook`:

```typescript
  getSmsAckResponse(): { contentType: string; body: string } {
    return {
      contentType: 'application/xml',
      body: '<?xml version="1.0" encoding="UTF-8"?><Response></Response>',
    };
  }
```

**Step 4: Add import for SmsMessage type**

Update line 12:

```typescript
import type { PhoneProvider, PhoneConfig, SmsMessage } from './types.js';
```

**Step 5: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: Type error only in phone-telnyx.ts now

**Step 6: Commit**

```bash
git add server/src/providers/phone-twilio.ts
git commit -m "feat(twilio): implement SMS send and webhook parsing"
```

---

### Task 3: Implement Telnyx SMS Methods

**Files:**
- Modify: `server/src/providers/phone-telnyx.ts`

**Step 1: Add sendSms method**

Add after `getStreamConnectXml` method (after line 189):

```typescript
  async sendSms(to: string, from: string, message: string): Promise<string> {
    if (!this.apiKey) {
      throw new Error('Telnyx not initialized');
    }

    const response = await fetch('https://api.telnyx.com/v2/messages', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        to,
        from,
        text: message,
      }),
    });

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Telnyx SMS failed: ${response.status} ${error}`);
    }

    const data = await response.json() as { data: { id: string } };
    return data.data.id;
  }
```

**Step 2: Add parseSmsWebhook method**

Add after `sendSms`:

```typescript
  parseSmsWebhook(body: string, contentType: string): SmsMessage | null {
    if (!contentType.includes('application/json')) {
      return null;
    }

    try {
      const event = JSON.parse(body);
      const eventType = event.data?.event_type;

      if (eventType !== 'message.received') {
        return null;
      }

      const payload = event.data?.payload;
      if (!payload) {
        return null;
      }

      return {
        from: payload.from?.phone_number || payload.from,
        to: payload.to?.[0]?.phone_number || payload.to,
        body: payload.text,
        messageId: event.data?.id,
      };
    } catch {
      return null;
    }
  }
```

**Step 3: Add getSmsAckResponse method**

Add after `parseSmsWebhook`:

```typescript
  getSmsAckResponse(): { contentType: string; body: string } {
    return {
      contentType: 'application/json',
      body: JSON.stringify({ status: 'ok' }),
    };
  }
```

**Step 4: Add import for SmsMessage type**

Update line 12:

```typescript
import type { PhoneProvider, PhoneConfig, SmsMessage } from './types.js';
```

**Step 5: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: No type errors

**Step 6: Commit**

```bash
git add server/src/providers/phone-telnyx.ts
git commit -m "feat(telnyx): implement SMS send and webhook parsing"
```

---

### Task 4: Add SMS State and Webhook Endpoint

**Files:**
- Modify: `server/src/phone-call.ts`

**Step 1: Add SMS state interface**

Add after `CallState` interface (around line 30):

```typescript
interface SmsState {
  sessionId: string;
  userPhoneNumber: string;
  resolve: (reply: string) => void;
  startTime: number;
}
```

**Step 2: Add SMS state tracking to CallManager**

Add after line 77 (`private currentCallId = 0;`):

```typescript
  private activeSmsSession: SmsState | null = null;
  private smsTimeoutMs = 30 * 60 * 1000; // 30 minutes default
```

**Step 3: Add /sms route to HTTP server**

In `startServer()`, add after the `/health` route (around line 97):

```typescript
      if (url.pathname === '/sms') {
        this.handleSmsWebhook(req, res);
        return;
      }
```

**Step 4: Add handleSmsWebhook method**

Add after `handleTelnyxWebhook` method (around line 423):

```typescript
  private handleSmsWebhook(req: IncomingMessage, res: ServerResponse): void {
    const contentType = req.headers['content-type'] || '';
    let body = '';

    req.on('data', (chunk) => { body += chunk; });
    req.on('end', () => {
      const message = this.config.providers.phone.parseSmsWebhook(body, contentType);

      if (message && this.activeSmsSession) {
        // Normalize phone numbers for comparison (remove +, spaces, dashes)
        const normalizePhone = (phone: string) => phone.replace(/[\s\-\+]/g, '');
        const messageFrom = normalizePhone(message.from);
        const expectedFrom = normalizePhone(this.activeSmsSession.userPhoneNumber);

        if (messageFrom === expectedFrom || messageFrom.endsWith(expectedFrom) || expectedFrom.endsWith(messageFrom)) {
          console.error(`[SMS] Received reply: ${message.body.substring(0, 50)}...`);
          this.activeSmsSession.resolve(message.body);
          this.activeSmsSession = null;
        } else {
          console.error(`[SMS] Ignoring message from ${message.from} (expected ${this.activeSmsSession.userPhoneNumber})`);
        }
      } else if (message) {
        console.error(`[SMS] Ignoring message - no active SMS session`);
      }

      const ack = this.config.providers.phone.getSmsAckResponse();
      res.writeHead(200, { 'Content-Type': ack.contentType });
      res.end(ack.body);
    });
  }
```

**Step 5: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: No type errors

**Step 6: Commit**

```bash
git add server/src/phone-call.ts
git commit -m "feat(sms): add SMS webhook endpoint and state tracking"
```

---

### Task 5: Add sendSmsAndWaitForReply Method

**Files:**
- Modify: `server/src/phone-call.ts`

**Step 1: Add sendSmsAndWaitForReply method**

Add after the new `handleSmsWebhook` method:

```typescript
  async sendSmsAndWaitForReply(message: string): Promise<string> {
    const sessionId = `sms-${Date.now()}`;
    console.error(`[${sessionId}] Sending SMS: ${message.substring(0, 50)}...`);

    // Send the SMS
    await this.config.providers.phone.sendSms(
      this.config.userPhoneNumber,
      this.config.phoneNumber,
      message
    );

    console.error(`[${sessionId}] SMS sent, waiting for reply...`);

    // Wait for reply with timeout
    return new Promise((resolve) => {
      this.activeSmsSession = {
        sessionId,
        userPhoneNumber: this.config.userPhoneNumber,
        resolve,
        startTime: Date.now(),
      };

      // Timeout after configured duration
      setTimeout(() => {
        if (this.activeSmsSession?.sessionId === sessionId) {
          console.error(`[${sessionId}] SMS reply timeout`);
          resolve('[No reply received - user did not respond within timeout period]');
          this.activeSmsSession = null;
        }
      }, this.smsTimeoutMs);
    });
  }
```

**Step 2: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: No type errors

**Step 3: Commit**

```bash
git add server/src/phone-call.ts
git commit -m "feat(sms): add sendSmsAndWaitForReply method"
```

---

### Task 6: Add Method Parameter to initiate_call Tool

**Files:**
- Modify: `server/src/index.ts`

**Step 1: Update tool schema**

In the `initiate_call` tool definition (around line 58), update `inputSchema.properties`:

```typescript
            properties: {
              message: {
                type: 'string',
                description: 'What you want to say to the user. Be natural and conversational.',
              },
              method: {
                type: 'string',
                enum: ['call', 'text'],
                description: 'Contact method: "call" for phone call, "text" for SMS. Defaults to CALLME_DEFAULT_METHOD env var or "call".',
              },
            },
```

**Step 2: Update tool handler**

Update the `initiate_call` handler (around line 112):

```typescript
      if (request.params.name === 'initiate_call') {
        const { message, method } = request.params.arguments as { message: string; method?: 'call' | 'text' };
        const contactMethod = method || process.env.CALLME_DEFAULT_METHOD || 'call';

        if (contactMethod === 'text') {
          const reply = await callManager.sendSmsAndWaitForReply(message);
          return {
            content: [{
              type: 'text',
              text: `SMS sent successfully.\n\nUser's reply:\n${reply}`,
            }],
          };
        }

        const result = await callManager.initiateCall(message);

        return {
          content: [{
            type: 'text',
            text: `Call initiated successfully.\n\nCall ID: ${result.callId}\n\nUser's response:\n${result.response}\n\nUse continue_call to ask follow-ups or end_call to hang up.`,
          }],
        };
      }
```

**Step 3: Verify types compile**

Run: `cd /mnt/volume_sfo2_01/code/call-me/.worktrees/sms-support/server && npx tsc --noEmit`
Expected: No type errors

**Step 4: Commit**

```bash
git add server/src/index.ts
git commit -m "feat(tool): add method parameter to initiate_call for SMS support"
```

---

### Task 7: Update README Documentation

**Files:**
- Modify: `README.md`

**Step 1: Add SMS configuration section**

Add after the "Optional Variables" table (around line 70):

```markdown
#### SMS Support (Optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `CALLME_DEFAULT_METHOD` | `call` | Default contact method: `call` or `text` |

When using SMS:
- Claude uses the same `initiate_call` tool with `method: "text"`
- User can override in prompts: "text me when done" or "call me about this"
- SMS conversations are interactive - Claude waits for your reply before continuing

**Twilio SMS Setup:**
- SMS uses the same phone number and credentials as voice calls
- Configure SMS webhook URL in Twilio Console → Phone Numbers → Your Number → Messaging → Webhook URL: `https://your-ngrok-url/sms`

**Telnyx SMS Setup:**
- SMS uses the same credentials as voice calls
- Configure Messaging Profile in Telnyx Portal with webhook URL: `https://your-ngrok-url/sms`
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add SMS configuration documentation"
```

---

### Task 8: Push and Test

**Step 1: Push all changes to fork**

```bash
git push csima sms-support
```

**Step 2: Manual testing checklist**

Test locally with Twilio or Telnyx:

1. Set `CALLME_DEFAULT_METHOD=text` in environment
2. Start the MCP server
3. Use Claude Code with a prompt that triggers `initiate_call`
4. Verify SMS is received on your phone
5. Reply to the SMS
6. Verify Claude receives your reply

Test override:
1. Set `CALLME_DEFAULT_METHOD=call`
2. Ask Claude to "text me when done"
3. Verify SMS is sent instead of call

**Step 3: Commit test results or fixes**

If any fixes needed, commit them with descriptive messages.

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Add SMS types to interface | types.ts |
| 2 | Implement Twilio SMS | phone-twilio.ts |
| 3 | Implement Telnyx SMS | phone-telnyx.ts |
| 4 | Add SMS webhook endpoint | phone-call.ts |
| 5 | Add sendSmsAndWaitForReply | phone-call.ts |
| 6 | Add method parameter to tool | index.ts |
| 7 | Update documentation | README.md |
| 8 | Push and test | - |
