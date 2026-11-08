# Setup auf dem MacBook

## 1. Voraussetzungen
- Homebrew: https://brew.sh
- Node.js 22 (mindestens 18, siehe `.nvmrc`):
  ```bash
  brew install nvm        # oder: brew install node@22
  nvm install && nvm use  # liest .nvmrc
  ```
- Git, Python 3 (nur für Frida): `brew install git python`

## 2. Repo holen
```bash
git clone https://github.com/PrevosHD/whalibmob.git
cd whalibmob
git checkout nando/linkcode-fix-username-handoff
```

## 3. Node-Abhängigkeiten
```bash
npm install
npm test          # erwartet: alle Tests grün
node cli.js       # CLI starten (oder: npx wa)
```
`package-lock.json` ist absichtlich nicht im Repo (Library-Konvention), `npm install` löst alles auf.
Optionale Pakete (`jimp`, `socks`, `qrcode-terminal`) werden mitinstalliert; schlägt eins fehl, läuft der Rest trotzdem.

## 4. Frida (für den Username-Mitschnitt)
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
frida --version
```
- **Android:** gerootet, passenden `frida-server` (gleiche Version wie `frida --version`) aufs Gerät, per USB/adb.
- **iPhone X (Jailbreak):** `frida-server` aus Sileo/Cydia (Repo `https://build.frida.re`), per USB.
- Test: `frida-ps -U` muss die Prozesse vom Gerät listen.

## 5. Sessions
Session-Daten (`whalibomb/`, `Sessions/`, `.waSession`) sind **nicht** im Repo.
Die 5 Nando-Nummern laufen auf dem Server weiter – auf dem Mac nichts davon neu registrieren.
Für den Username-Mitschnitt eine **Wegwerf-Testnummer** in der offiziellen App benutzen.

## 6. Weiter mit Claude Code
```bash
claude
> lies docs/HANDOFF-username.md und SETUP-MAC.md
```
