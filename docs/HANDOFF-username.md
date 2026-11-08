# Übergabe: whalibomb / Nando-Bots / Username (Stand 08.11.2026)

## Setup
- Server: `/home/whatsapp/whalibomb/whalibmob` (whalibmob v5.34.0, CLI `cli.js`)
- Primary-Stores der Nummern: `whalibomb/whalibmob/whalibomb/Sessions/<nummer>/<nummer>.json`
- kebab-Bots (Companion, Baileys 7.0.0-rc13) in pm2, Sessions in `/home/whatsapp/kebab_sessions/<Bot>`

- Bots: Nando_05, Nando_20, Nando_25, Nando_62, Nando_76 (Nummern siehe Server-Datei `/home/whatsapp/whalibomb/HANDOFF-username.md`)

## Erledigt: Bots konnten keine Nachrichten entschlüsseln
- Ursache: `lib/PrimaryPairing.js` → `allocateCompanionDeviceIndex()` hat beim Linkcode-Pairing
  eine signierte ADV-Key-Index-List mit **leeren** `validIndexes` geschickt (neuer Index fehlte).
  Offizielle Clients (Handy/WA Web) verwerfen dann das Companion-Gerät und verschlüsseln nicht dafür
  → `No session found to decrypt message` / leere Messages in kebab. Baileys-Bots prüfen ADV nicht,
  darum sahen die das Gerät trotzdem.
- Fix: `validIndexes: existing.concat([next])` (statt `existing.slice()`).
- Alle 5 neu per Linkcode verbunden → jetzt Gerät `:2`, `linkedDeviceIndexes: [1, 2]`.
- Ergebnis: Nando_05/20/25/76 entschlüsseln wieder.
- Offen: **Nando_62** bekommt von Nandos WA Web (Gerät `:26`) noch keinen Sender-Key
  (vom Handy `:0` schon). → WA Web neu laden (F5), dann in der Gruppe schreiben und Logs prüfen.
- DECRYPT_ERRORs in der whalibmob-Konsole (Primary) sind für kebab egal (Primary war offline,
  PreKeys out of sync).

## Username setzen in whalibmob (`/username <name>`) — Request gefangen + implementiert

Usernames (WhatsApp, seit 29.06.2026): 3–35 Zeichen, a-z 0-9 `.` `_`, optional „Username-Key" (PIN).
`/name` setzt nur den Anzeigenamen (pushname), nicht den Username.

**Kernbefund:** Usernames laufen über **w:mex (Meta GraphQL)**, NICHT über klassisches
Token-XML (`lib/tokens.js` braucht daher kein neues Token). Beide Calls sind
`iq type=get xmlns=w:mex` und werden nur per `query_id` unterschieden. Mitgeschnitten vom
offiziellen iOS-Client (App 26.41.74) per Frida auf dem iPhone X:

- **Check** `UsernameCheckQueryV2`, `query_id=26960756810228820`,
  `variables={"username":"<name>","session_id":"<uuid>","source":"USER_INPUT","include_suggestions":false,"_skip_concrete_typename_fields":true}`
- **Set** `UsernameSetMutation`, `query_id=27363455153316957`,
  `variables={"session_id":"<uuid>","reserved":true,"input":"<name>","source":"USER_INPUT","pin":null,"_skip_concrete_typename_fields":true}`
  (Username steht in `input`, nicht `username`. PIN separat: `UsernamePINSetMutation`.)
  Der offizielle Client nutzt eine `session_id` (UUID) für Check und Set derselben Bearbeitung.

**Implementiert (2026-11-08):**
- `lib/Client.js`: `checkUsername(name[, sessionId])` und `setUsername(name[, {sessionId, pin}])`,
  beide über das vorhandene `_mexQuery(queryId, variables)`. `checkUsername` ist read-only,
  `setUsername` ändert echt. Beide geben `{ data, sessionId }` zurück.
- `cli.js`: `/username <name>` (setzen), `/username check <name>` (nur prüfen), in Help unter „Account".
- `index.d.ts`: Deklarationen ergänzt, `npm run test:types` grün.

**Noch offen — Live-Test gegen Server (auf dem Server fortsetzen):**
1. Mit einer verbundenen whalibmob-Session zuerst `/username check <name>` (ungefährlich) —
   bestätigt Wire-Format + Verfügbarkeits-Antwort.
2. Dann `setUsername` an einer **Wegwerf-Testnummer** (NICHT eine der 5 Nummern), danach Nando_20.
3. `data`-Antwortstruktur prüfen (Feldnamen der GraphQL-Response, z.B. Erfolg/Fehlercode) und
   bei Bedarf einen `dataPath`/Erfolgs-Parse in `setUsername`/`checkUsername` ergänzen.

Frida-Details, Device-Zugang und Capture-Skripte: siehe Memory `username-mex-capture.md`
(Skripte im Scratchpad: `capture9.py` = leichter w:mex-Hook, der die Mutation fing).

### Geräte
- Android (Root + frida-server) → geht.
- iPhone X mit Jailbreak → geht ebenfalls (frida-server über Sileo/Cydia). iPhone X kann max. iOS 16;
  vorher prüfen, ob die aktuelle WhatsApp-Version dort noch läuft und Usernames anbietet.

## Hinweise
- Keine WhatsApp-Neuregistrierung der 5 Nummern – nur Linkcode neu, falls nötig.
