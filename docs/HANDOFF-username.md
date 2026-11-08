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

## Offen: Username setzen in whalibmob (Ziel: `/username <name>` in der CLI)
- Usernames (WhatsApp, seit 29.06.2026): 3–35 Zeichen, a-z 0-9 `.` `_`, optional „Username-Key“ (PIN).
- whalibmob kann Usernames nur **lesen** (`lib/WAUSync/Protocols/USyncUsernameProtocol.js`,
  `contact_update`, Gruppen-Attribute `participant_username` …). Kein Set-Request vorhanden,
  `lib/tokens.js` enthält kein Username-Token. Baileys kann es auch nicht. Kein öffentliches
  Projekt (whatsmeow etc.) hat den Set-Request dokumentiert.
- `/name` setzt nur den Anzeigenamen (pushname), nicht den Username.

### Plan: Request vom offiziellen Client mitschneiden
1. Gerät mit offizieller WhatsApp + **Wegwerf-Testnummer** (NICHT eine der 5 Nummern).
2. Frida-Hook auf den Stanza-Encoder (BinaryNode/XMPP-Node vor Noise-Verschlüsselung),
   alle ausgehenden `<iq>` loggen.
3. In der App Username setzen (und ggf. Username-Key) → mitgeschnittene IQs sichern.
   Zusätzlich: Username-Verfügbarkeit prüfen (beim Tippen) mitschneiden.
4. Daraus `Client.setUsername()` / `checkUsername()` + CLI-Befehl `/username` bauen,
   erst an der Testnummer testen, dann Nando_20.
- Vorhandene Frida-Infrastruktur in whalibmob: `frida/android`, `frida/ios` (bisher nur
  Registrierung/Attestation) – als Ausgangspunkt für Gerät-Anbindung nutzen.

### Geräte
- Android (Root + frida-server) → geht.
- iPhone X mit Jailbreak → geht ebenfalls (frida-server über Sileo/Cydia). iPhone X kann max. iOS 16;
  vorher prüfen, ob die aktuelle WhatsApp-Version dort noch läuft und Usernames anbietet.

## Hinweise
- Keine WhatsApp-Neuregistrierung der 5 Nummern – nur Linkcode neu, falls nötig.
