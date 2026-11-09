# Session Log — 2026-11-08/09

What was investigated and achieved in this session. Phone numbers are masked
(`+4915xxxxx`, `+63xxxx`); usernames, GraphQL query ids and group/device ids are kept.

---

## 1. WhatsApp username set — reverse engineered and implemented

**Goal:** add the ability to set a WhatsApp username from whalibmob (it could only read them before).

**Captured** (Frida on a jailbroken iPhone, hooking `-[XMPPConnection sendStreamElement:…]`):
usernames go over `w:mex` GraphQL. A username save is exactly two stanzas sharing one `session_id`:
- check — `UsernameCheckQueryV2`, `query_id 26960756810228820`, var `username`.
- set — `UsernameSetMutation`, `query_id 27363455153316957`, var `input`, `reserved:true`, `pin:null`.

A full-stanza capture (all outgoing xmlns) of one save confirmed there is **nothing else** — no
attestation IQ, no companion step.

**Implemented** (pushed):
- `Client.checkUsername`, `Client.setUsername` (runs the check first, reuses the `session_id`),
  `Client.lookupUsername` (username → jid via usync).
- Argo reply reader (`argoInlineStrings` / `interpretUsernameReply`) — the replies are Argo that
  the generic decoder collapses to `false`; the enum labels (SUCCESS / ALREADY_TAKEN / INVALID /
  Bad Request / Internal Server Error / State Already Exists) are read inline from the buffer.
- CLI: `/username <name>`, `/username check <name>`, `/username whois <name>`, `/username get [jid]`,
  and `/mexraw <query_id> <json>` (diagnostic raw mex sender).

**Result — it does NOT persist from a library client.** The stanza is byte-identical to the official
app, yet the server only *provisions* a username for attested official-app connections. From
whalibmob the set returns a hollow `SUCCESS` and the write is silently dropped. Proven by an A/B on
the same server: a name set by the official app reads `ALREADY_TAKEN`, a name set by whalibmob reads
`available`. Ruled out: wire format, body `queryId`, `reserved` flag, `session_id` reuse, announced
version (`2.26.43.75` iOS, current), async notification, a missing GraphQL op, region (a `+4915xxxxx`
account via whalibmob fails too), and account age. The only remaining difference is connection/account
trust (App Attest / attested registration origin) — **not fixable by stanza replication.**

Full technical writeup: `docs/USERNAME-SET-REVERSE-ENGINEERING.md`. Status in
`docs/HANDOFF-username.md`.

**Still usable:** `checkUsername` (correct global availability), `lookupUsername`, the Argo reader.
**Dead:** `setUsername` (correct but no effect), `fetchUsername` (phone→username is withheld by usync).

---

## 2. Account receive/health diagnosis

**Nando_25 (`+63xxxx`) sends but receives nothing (1 tick at every sender).** Diagnosed without
cold-sending, by reading the send-side device fan-out log `DM_SEND to=… otherJids=[…]`:
- healthy recipient → device-suffixed jids, e.g. `…@s.whatsapp.net, …:26@s.whatsapp.net`.
- Nando_25 → only the bare `…@s.whatsapp.net`, **no `:N` device**.

So the account has **no device in WhatsApp's server-side device list** → no delivery target → every
sender (official app included) gets 1 tick, nothing is delivered, while the whalibmob session can
still authenticate and send. Only re-registration (SMS OTP) repopulates the device list — not fixable
client-side. The same class explains another account that "only works in groups, not DMs".

**Bans.** `+63xxxx` flagged bot numbers get stripped/banned on starting new chats. During this session:
- Nando_20 (`+63xxxx`) — spam-banned (403, "primary device is gone", session revoked).
- Nando_437 (`+4915xxxxx`) — banned (403) **by a diagnostic cold `/send` to a new number**. Lesson
  recorded: never cold-send from these accounts for diagnostics; reads and existing-chat fan-out are safe.

**Cleanup.** Removed the dead aliases from whalibmob `/list`: Nando_20, Nando_437, Nando_25.
Remaining active accounts: Nando_05, Nando_62, Nando_76. Session folders for the removed numbers were
left on disk.

---

## 3. Group decryption / "Waiting for this message" (open)

The CLI logs flooded `DECRYPT_ERROR from 120363039259018408@g.us : No session found to decrypt
message` (same ids repeating — a retry loop that never heals). whalibmob *does* send retry receipts on
group decrypt failure and *does* handle incoming ones, but the heal does not complete: in the group,
messages from `+4915xxxxx` (real phones) decrypt fine while messages from `+63xxxx` (whalibmob bots)
do not, and a companion linked by QR shows "Waiting for this message" for them.

This matches open Baileys issues:
- [#2884](https://github.com/WhiskeySockets/Baileys/issues/2884) — companion cannot recover inbound
  group sender keys after re-link; retry receipts sent and ignored; no heal path.
- [#2730](https://github.com/WhiskeySockets/Baileys/issues/2730) — linked device, sender-key fetch 404.

**Fix direction (not implemented):** on a retry receipt / phash mismatch, re-usync the group
participants' device lists to discover newly-linked companion devices and re-distribute the
SenderKeyDistributionMessage to the requesting device, rather than only re-sending cached ciphertext.
Mitigations: unlink+relink the companion; keep the sending primaries online; send once into the group
from each bot to force sender-key redistribution.

---

## Next session

Numbers are resting until ~late evening 2026-11-10, then experiments continue.
