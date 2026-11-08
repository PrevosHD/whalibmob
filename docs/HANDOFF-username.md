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

**ENDERGEBNIS (2026-11-09): `setUsername` persistiert NICHT über whalibmob — server-seitiges Client-Trust-Gating, kein Code-/Protokoll-Problem.**

Der Set ist korrekt implementiert und byte-identisch zum offiziellen Client (voller Frida-Mitschnitt
eines App-Saves = genau Check + eine `UsernameSetMutation`, sonst nichts — kein Attestation-IQ,
kein Nicht-w:mex-Begleitschritt). Trotzdem:
- Offizielle App (+49-Gerät): Set persistiert (`NandoBocchi_580` → serverweit `ALREADY_TAKEN`).
- whalibmob (getestet +49 Nando_437, +63 Nando_20 und frisch +63 Nando_05): Set antwortet `SUCCESS`,
  aber Re-Check zeigt den Namen weiter als `available` — der Write wird still verworfen.
- A/B am selben Server: Device-Name `ALREADY_TAKEN`, whalibmob-Name `available`.

Ausgeschlossen: Wire-Format, body-`queryId`, `reserved`-Flag, `session_id`-Reuse (Check→Set),
angekündigte Version (aktuell `2.26.43.75` iOS), async Notification nach Set, fehlende GraphQL-Op
(Binary hat nur `UsernameSetMutation`/`UsernameCheckQueryV2`/`UsernameGetQuery`/PIN — keine separate
Activation/Reserve-Mutation; `UsernameActivationObserving` ist nur ein Client-Observer), Region
(+49 via whalibmob scheitert ebenso), Account-Alter/Rate-Limit (etablierter Nando_20 scheitert auch).
Übrig bleibt nur der **Verbindungs-/Account-Trust**: der offizielle Client ist hardware-attestiert
(App Attest / attestierter Registrierungs-Ursprung), whalibmob nicht — der Server akzeptiert die
Mutation syntaktisch (`SUCCESS`), provisioniert sie aber nur für vertrauenswürdige Clients.
**Nicht durch Stanza-Replikation lösbar.**

**Was funktioniert & bleibt nutzbar:**
- `checkUsername(name)` — liefert korrekt globale Verfügbarkeit. Enum (in `Query` gewrappt):
  `SUCCESS`=frei, `INVALID`=ungültiger Name, `ALREADY_TAKEN`=vergeben. CLI `/username check <name>`.
- `lookupUsername(name)` — usync Username→JID (Server echot Input als `type="out"`). CLI `/username whois`.
- Argo-Reader (`argoInlineStrings`/`interpretUsernameReply` in `lib/Client.js`) liest die
  self-describing Replies (SUCCESS/ALREADY_TAKEN/Bad Request/State Already Exists/Internal Server Error).
- `/mexraw <qid> <json>` (Diagnose), `_mexQuery(…, {wantArgoRaw, body})`.

**Tote Enden / optionale Aufräumung:**
- `setUsername` sendet korrekt, hat aber ohne offizielle-App-Attestierung keinen Effekt — als solches
  dokumentiert lassen oder entfernen.
- `fetchUsername` (phone→username) ist faktisch tot: usync hält diese Richtung aus Privacy-Gründen
  zurück (liefert immer `null`).

Frida-Details, Device-Zugang und Capture-Skripte: siehe Memory `username-mex-capture.md`
(Scratchpad: `capture9/10/11.py` = w:mex- bzw. Voll-Stanza-Hooks).

### Geräte
- Android (Root + frida-server) → geht.
- iPhone X mit Jailbreak → geht ebenfalls (frida-server über Sileo/Cydia). iPhone X kann max. iOS 16;
  vorher prüfen, ob die aktuelle WhatsApp-Version dort noch läuft und Usernames anbietet.

## Hinweise
- Keine WhatsApp-Neuregistrierung der 5 Nummern – nur Linkcode neu, falls nötig.
