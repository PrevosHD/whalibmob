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

**Live getestet (2026-11-08, Testnummer +63…420) — funktioniert:**
- `/username check <name>` und `/username <name>` erzeugen Stanzas identisch zum iOS-Client,
  Server verarbeitet beide.
- Antworten kommen als **argo** (`<result format="argo">`). `decodeArgo` macht daraus ein
  bloßes `false`; die eigentlichen Enum-Labels stehen aber inline als längenpräfigierte
  UTF-8 im Puffer (`(len<<1)` + Bytes). `argoInlineStrings()` + `interpretUsernameReply()`
  in `lib/Client.js` ziehen sie raus:
  - Set erfolgreich → `04 0e SUCCESS …` → `{ ok:true, status:'SUCCESS' }` (Name wurde gesetzt).
  - Check vergeben → `… Query … INVALID … ALREADY_TAKEN …` → `{ available:false, status:'ALREADY_TAKEN' }`.
  - Set abgelehnt → `… "Bad Request" … CRITICAL …` → `{ ok:false, status:'Bad Request' }`
    (z.B. unerlaubtes `-`; Zeichen werden jetzt zusätzlich client-seitig per Regex abgefangen).
- `checkUsername` gibt `{ available, status, markers, data, sessionId }`,
  `setUsername` gibt `{ ok, status, markers, data, sessionId }`.

**Bestätigt funktionsfähig (2026-11-09):** Ein Username wird nur per **Check→Set mit derselben
`session_id`** gesetzt. `setUsername` macht jetzt zuerst `checkUsername` und übernimmt dessen
`session_id`. Ein Set „aus dem Nichts" (frische, ungeprüfte `session_id`) antwortet
`Internal Server Error` / CRITICAL. `Bocchi63420` wurde so erfolgreich gesetzt → Handle `@bocchi63420`.
Enum-Labels:
- Check (in `Query` gewrappt): `SUCCESS`=frei, `INVALID`=ungültiger Name, `ALREADY_TAKEN`=vergeben.
- Set: `SUCCESS`=gesetzt; Fehler `State Already Exists` (Name schon reserviert),
  `Bad Request` (unerlaubtes Zeichen), `Internal Server Error` (ungeprüfte Session).

**Noch offen:**
1. An Nando_20 testen (erst mit `/username check <name>` einen freien Namen finden).
2. Optional sauber: `decodeArgo` (`lib/argo`) so erweitern, dass es diese self-describing
   Replies vollständig als Objekt dekodiert, statt der inline-String-Heuristik.

Frida-Details, Device-Zugang und Capture-Skripte: siehe Memory `username-mex-capture.md`
(Skripte im Scratchpad: `capture9.py` = leichter w:mex-Hook, der die Mutation fing).

### Geräte
- Android (Root + frida-server) → geht.
- iPhone X mit Jailbreak → geht ebenfalls (frida-server über Sileo/Cydia). iPhone X kann max. iOS 16;
  vorher prüfen, ob die aktuelle WhatsApp-Version dort noch läuft und Usernames anbietet.

## Hinweise
- Keine WhatsApp-Neuregistrierung der 5 Nummern – nur Linkcode neu, falls nötig.
