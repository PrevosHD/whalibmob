# WhatsApp Username Set — Reverse Engineering Notes

Status: **the set request is fully reverse engineered and reproduced byte-for-byte,
but it does not persist from a library client.** WhatsApp gates the actual
provisioning of a username behind the official, hardware-attested app connection.
From whalibmob the server answers `SUCCESS` and then silently drops the write.

This document records how the request was captured, the complete stanzas, the
server replies, the CLI experiments, and the evidence for the conclusion. It may
be useful to the whalibmob author and to anyone else looking at WhatsApp usernames.

Phone numbers in logs are masked (`+4915xxxxx`, `+63xxxx`). Usernames and GraphQL
query ids are kept — they are the point.

---

## 1. Background

WhatsApp usernames (rolled out from 2026-06-29) are 3–35 chars, `a-z A-Z 0-9 . _`.
whalibmob could already **read** usernames (usync username protocol, group
`participant_username`, contact updates) but had no way to **set** one, and
`lib/tokens.js` has no username token. No public project (Baileys, whatsmeow, …)
had documented the set request.

Key early finding: **usernames do not go over classic token-XML. They go over
`w:mex` (Meta GraphQL tunnelled over the XMPP/noise connection).** Both the
availability check and the set are `iq type=get xmlns=w:mex`, told apart only by
the GraphQL `query_id`.

---

## 2. Capture setup (Frida on a jailbroken iPhone)

- Device: iPhone X, iOS 16.7.14, rootless jailbreak (`/var/jb`), WhatsApp `26.41.74`
  (`net.whatsapp.WhatsApp`). Logged into a throwaway test number (`+4915xxxxx`),
  never a production number.
- `frida-server` 17.x from `build.frida.re` (Sileo). It binds `127.0.0.1:27042`,
  so reach it from the Mac through an SSH local forward:
  `ssh -f -N -L 27042:127.0.0.1:27042 mobile@<device>` then `frida -H 127.0.0.1:27042`.
- Mac-side Frida **major version must match** the device (17).

### Frida gotchas that cost time

- `ApiResolver("objc")` **crashes WhatsApp** (`EXC_BAD_ACCESS` in
  `swift_getTypeByMangledName` on the gum-js thread). Use the **ObjC runtime C
  API** instead (`objc_getClass`, `class_copyMethodList`, `method_getImplementation`,
  `object_getClass`, `sel_registerName`, `objc_msgSend`).
- Enumerating all classes (`objc_copyClassList` + per-class method walk) trips a
  breakpoint/`class_getName` trap on Swift metaclasses. Restrict to named classes.
- Deep, synchronous work inside the interceptor on the send thread trips the iOS
  watchdog (`0x8BADF00D`) on this slow device. Keep the hook cheap.

### What is hooked

All outgoing stanzas go through
`-[XMPPConnection sendStreamElement:withMaybeTimeout:fromExclusiveSender:ignoreExclusiveSender:completion:]`.
The stanza is an `XMPPIQStanza` whose `description` is redacted, so the tree is
walked via `name`, `attributesStringDictionary`, `children`, `childCount`,
`objectAtIndex:`, `dataValue`, `elementValue`, `xmlns`.

---

## 3. The captured request (complete)

Changing the username in the official app sends **exactly two** `w:mex` stanzas,
in order, sharing one `session_id`. A full-stanza capture (every outgoing xmlns,
not just `w:mex`) of one save confirmed there is **nothing else** — no attestation
IQ, no second mutation, no non-`w:mex` companion step:

```
#1 iq xmlns=w:mex type=get kids=[query]     <- availability check
#2 iq xmlns=w:mex type=get kids=[query]     <- set mutation
```

### 3.1 Availability check — `UsernameCheckQueryV2`, query_id `26960756810228820`

```xml
<iq to="s.whatsapp.net" type="get" xmlns="w:mex">
  <query query_id="26960756810228820">
    {"queryId":"26960756810228820",
     "variables":{"username":"NandoBocchi_58",
                  "session_id":"90B1CA8D-6DCD-4B49-A890-1F61D6387582",
                  "source":"USER_INPUT",
                  "include_suggestions":false,
                  "_skip_concrete_typename_fields":true}}
  </query>
</iq>
```

### 3.2 Set / reserve — `UsernameSetMutation`, query_id `27363455153316957`

```xml
<iq to="s.whatsapp.net" type="get" xmlns="w:mex">
  <query query_id="27363455153316957">
    {"queryId":"27363455153316957",
     "variables":{"session_id":"90B1CA8D-6DCD-4B49-A890-1F61D6387582",
                  "reserved":true,
                  "input":"NandoBocchi_58",
                  "source":"USER_INPUT",
                  "pin":null,
                  "_skip_concrete_typename_fields":true}}
  </query>
</iq>
```

Notes:
- The username is in `input` for the set (not `username`), and in `username` for the check.
- `reserved:true`, `pin:null` (a separate `UsernamePINSetMutation` exists for the username PIN).
- The set reuses the **same `session_id`** as the preceding check. A set with a
  fresh, never-checked `session_id` is answered `Internal Server Error` / `CRITICAL`,
  so whalibmob's `setUsername` runs the check first and carries the id over.

### 3.3 Other username GraphQL ops found in the binary

`UsernameCheckQueryV2`, `UsernameCheckMultiQueryV2`, `UsernameSetMutation`,
`UsernameGetQuery`, `UsernameGetCountryCodeQuery`, `UsernamePINGetQuery`,
`UsernamePINSetMutation`, and incoming notifications
(`UsernameSetNotification`, `UsernameUpdateNotification`, `UsernameDeleteNotification`,
`AccountSyncUsernameNotification`). **There is no separate activation / reserve
mutation** — `UsernameActivationObserving` is a client-side observer protocol, not
a server operation.

---

## 4. The replies are Argo

`w:mex` replies come back as `<result format="argo">` (Meta's compact Argo
encoding), which the generic decoder in whalibmob collapses to a bare `false`. The
enum labels, however, sit inline in the Argo buffer as length-prefixed UTF-8
(`(len << 1)` then the bytes), so they can be read directly without the schema.
Observed replies (hex → inline labels):

| Context | Argo (hex) | Labels | Meaning |
|---|---|---|---|
| check, taken | `040a51756572790e494e56414c49441a414c52454144595f54414b454e14…` | `Query`,`INVALID`,`ALREADY_TAKEN` | not available |
| check, invalid name | `040a51756572790e494e56414c494412000a010e0000000303` | `Query`,`INVALID` | invalid |
| check, free | `040a51756572790e5355434345535312000a010e0101010303` | `Query`,`SUCCESS` | available |
| set, ok | `040e535543434553530c00000e010303` | `SUCCESS` | accepted |
| set, bad char | `046242616420526571756573…435249544943414c…` | `Bad Request`,`error_code`,`is_retryable`,`severity`,`CRITICAL` | rejected |
| set, internal error | `0476496e7465726e616c20536572766572204572726f72…435249544943414c…` | `Internal Server Error`,…,`CRITICAL` | rejected |
| set, already reserved | `0474537461746520416c726561647920457869737473…435249544943414c…` | `State Already Exists`,…,`CRITICAL` | rejected |

A small helper (`argoInlineStrings` / `interpretUsernameReply` in `lib/Client.js`)
extracts these labels; the full Argo decoder was intentionally not touched.

---

## 5. whalibmob implementation

- `Client.checkUsername(name[, sessionId])` → `{ available, status, markers, data, sessionId }`
- `Client.setUsername(name[, {sessionId, pin}])` → `{ ok, status, markers, data, sessionId }`
  (runs the check first and reuses the `session_id`)
- `Client.lookupUsername(name)` → `{ jid, onWhatsApp }` (usync, contact protocol by username)
- CLI: `/username <name>`, `/username check <name>`, `/username whois <name>`,
  `/username get [jid]`, and `/mexraw <query_id> <json-body>` (diagnostic raw mex sender).

The stanza whalibmob sends is identical to the app's (same `query_id`, same
variable shape, same `session_id` reuse). The only cosmetic difference — whalibmob
sent `{"variables":…}` without the redundant top-level `queryId` — was tested and
makes no difference (the server keys off the `<query query_id>` attribute).

---

## 6. CLI logs (phone numbers masked)

### 6.1 A successful-looking set that does not persist (fresh +49 account)

```
wa +4915xxxxx> /username check nando437bocchi
checking...
wa +4915xxxxx> available: yes  (SUCCESS, 425ms)

wa +4915xxxxx> /username nando437bocchi
setting...
   <iq id="…" to="s.whatsapp.net" type="get" xmlns="w:mex">
     <query query_id="26960756810228820">{"variables":{"username":"nando437bocchi","session_id":"…","source":"USER_INPUT","include_suggestions":false,"_skip_concrete_typename_fields":true}}</query>
   <iq id="…" to="s.whatsapp.net" type="get" xmlns="w:mex">
     <query query_id="27363455153316957">{"variables":{"session_id":"…","reserved":true,"input":"nando437bocchi","source":"USER_INPUT","pin":null,"_skip_concrete_typename_fields":true}}</query>
wa +4915xxxxx> username set to "nando437bocchi"  (SUCCESS, 1185ms)

wa +4915xxxxx> /username check nando437bocchi
checking...
wa +4915xxxxx> available: yes  (SUCCESS, 207ms)      <-- STILL AVAILABLE: not persisted
```

### 6.2 The A/B that proves it is client-gating, not protocol

Same server, same `checkUsername` path, run back to back. `NandoBocchi_580` was set
on the **official app** (a `+4915xxxxx` device); `nando437dbg` was set via
**whalibmob**:

```
wa +4915xxxxx> /username check NandoBocchi_580
wa +4915xxxxx> available: no   (ALREADY_TAKEN, 70ms)   <-- official-app set persisted
wa +4915xxxxx> /username check nando437dbg
wa +4915xxxxx> available: yes  (SUCCESS, 72ms)          <-- whalibmob set dropped
```

### 6.3 Nothing comes back after the set

With full debug on, after a whalibmob set there is no asynchronous username
notification or error — only keepalive pings. The write is simply not applied.

---

## 7. What was ruled out

The set does not persist from whalibmob, and every protocol-level explanation was
tested and eliminated:

- **Wire format** — identical to the app (full capture = check + one mutation, nothing else).
- **Body `queryId`** — adding the redundant top-level `queryId` changes nothing.
- **`session_id` binding** — check-then-set with a shared `session_id` is done; a
  fresh id gives `Internal Server Error`, a reused one gives `SUCCESS` (but still no persist).
- **Announced version** — whalibmob announces `2.26.43.75` / iOS, which is current
  and username-capable (not a 405).
- **A missing step** — the binary has no activation/reserve mutation; a full-stanza
  capture shows no companion IQ.
- **Region** — a `+4915xxxxx` account via whalibmob fails just like `+63xxxx` ones,
  while the `+4915xxxxx` official-app device succeeds.
- **Account age / rate limit** — an established account and a freshly registered one
  both fail the same way.

## 8. Conclusion

The only remaining difference between the official app and whalibmob is **connection /
account trust**. The official iOS client presents a hardware-backed attestation
(App Attest) and an attested registration origin that whalibmob cannot reproduce.
WhatsApp accepts the username mutation syntactically (`SUCCESS`) but only
**provisions** it for trusted clients, and silently drops it otherwise — classic
anti-abuse shadowing (do not tell the bot it failed).

This is **not** fixable by replicating the stanza, because the stanza is already
identical. It would require satisfying the connection-level attestation, which is
outside what a library client can do.

### Still useful

- `checkUsername` works and reflects global state (`ALREADY_TAKEN` / available / `INVALID`).
- `lookupUsername` (username → jid) works via usync.
- The Argo label reader works for all observed replies.

### Dead ends

- `setUsername` sends correctly but has no effect without official-app attestation.
- `fetchUsername` (phone → username) always returns `null`: usync withholds that
  direction for privacy.

---

*Captured and verified 2026-11-08/09. Test numbers only; no production account was
re-registered.*
