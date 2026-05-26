# MEX — WhatsApp's Internal GraphQL Protocol

Baron-Baileys-v2 exposes all 208 WhatsApp MEX operations as first-class socket methods. MEX is WhatsApp's internal GraphQL system that runs over the existing WebSocket connection — no extra HTTP requests, no authentication tokens needed beyond your normal session.

## What is MEX?

MEX (short for **M**obile **Ex**ecution) is WhatsApp's internal GraphQL-over-WebSocket protocol. Instead of HTTP, queries are sent as XML IQ stanzas over the same WebSocket used for messages. The server replies in the same stanza with a JSON response body.

Internally WhatsApp uses Pando/MEX — a Facebook-originated mobile GraphQL framework. Every feature from privacy settings to newsletter management to passkeys goes through MEX.

## How to Use

Every socket returned by `makeWASocket()` already has all MEX methods available — no extra configuration needed.

```js
import makeWASocket from 'baron-baileys-v2'

const sock = makeWASocket({ auth: state })

// Example: fetch your privacy settings
const settings = await sock.getPrivacySettings(sock.user.id)

// Example: set last seen to contacts only
await sock.setPrivacySetting('LAST_SEEN', 'CONTACTS')

// Example: update your text status (About)
await sock.updateTextStatus('Hey there! I am using WhatsApp.')
```

## What MEX Methods Are There?

MEX methods are grouped by feature area and live in separate socket files:

| Area | Methods | File |
|------|---------|------|
| Privacy & Profile | 26 methods | [PRIVACY.md](PRIVACY.md) |
| Registration & Passkeys | 84 methods | [REGISTRATION.md](REGISTRATION.md) |
| Managed Accounts & Payments Passkey | 27 methods | [MANAGED-ACCOUNT.md](MANAGED-ACCOUNT.md) |
| Interoperability (3rd-party messaging) | 7 MEX + 9 IQ methods | [INTEROP.md](INTEROP.md) |
| Newsletters | ~40 methods | see `src/Socket/newsletter.js` |
| Groups | ~15 methods | see `src/Socket/groups.js` |
| Username | 6 methods | [USERNAME.md](USERNAME.md) |
| Communities | ~10 methods | see `src/Socket/communities.js` |

## Error Handling

All MEX methods throw a `Boom` error on failure:

```js
import { Boom } from '@hapi/boom'

try {
    await sock.setPrivacySetting('LAST_SEEN', 'NONE')
} catch (err) {
    if (err instanceof Boom) {
        console.log('Status code:', err.output.statusCode)
        console.log('Message:', err.message)
    }
}
```

Common status codes:

| Code | Meaning |
|------|---------|
| 400 | Invalid variables or unexpected server response |
| 401 | Not authenticated |
| 404 | Resource not found |
| 429 | Rate limited |

## Technical Details

Each MEX call sends this IQ stanza over WebSocket:

```xml
<iq id="<tag>" type="get" to="s.whatsapp.net" xmlns="w:mex">
  <query query_id="<numeric_doc_id>">{"variables":{...}}</query>
</iq>
```

The `query_id` is a numeric ID (16–17 digits) that identifies the specific GraphQL operation on WhatsApp's servers. These IDs come from the asset files in `assets/whatsapp-android-mex_client_persist_ids.json`, which are extracted from the WhatsApp APK.

All 208 operations from WhatsApp are implemented. IDs were verified against the APK file.
