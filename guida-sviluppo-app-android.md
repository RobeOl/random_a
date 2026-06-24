# Guida: Da app HTML/JS ad APK Android via GitHub Actions

Questa guida descrive come trasformare un'applicazione web (HTML + JavaScript + CSV)
in un'app Android installabile, usando **Capacitor** come wrapper nativo e
**GitHub Actions** per la compilazione in cloud — senza bisogno di Android Studio
in locale (funziona anche su macOS Big Sur o versioni precedenti).

---

## Prerequisiti

Sul Mac è sufficiente avere installato:

- **Node.js** v20 o superiore (verifica con `node -v`)
- **npm** (incluso con Node.js)
- **Git** (verifica con `git --version`)
- Un account **GitHub** con SSH configurato

Per configurare SSH su GitHub (una tantum):

```bash
# Genera una chiave SSH (se non ne hai già una)
ssh-keygen -t ed25519 -C "tua-email@esempio.com"

# Copia la chiave pubblica
cat ~/.ssh/id_ed25519.pub
```

Incolla la chiave su **GitHub → Settings → SSH and GPG keys → New SSH key**.
Verifica che funzioni con: `ssh -T git@github.com`

---

## Struttura del progetto

```
esperto-alberi-app/
├── www/                            ← sorgente web (l'unica cartella da modificare)
│   ├── index.html                  ← la tua app HTML (rinominata da esperto_alberi.html)
│   ├── alberi.csv
│   └── immagini/
│       ├── foglie/
│       ├── foglie_dettaglio/
│       ├── tronchi/
│       ├── frutti/
│       └── chiome/
├── android/                        ← progetto nativo Android (generato da Capacitor)
│   ├── app/
│   │   ├── build.gradle            ← configurazione build Android
│   │   └── src/main/res/mipmap-*/ ← icone dell'app
│   └── keystore/
│       └── debug.keystore          ← certificato di firma fisso
├── resources/
│   └── icon.png                    ← icona sorgente (1024x1024, opzionale)
├── capacitor.config.json           ← configurazione Capacitor
├── package.json
└── .github/
    └── workflows/
        └── build-apk.yml           ← workflow CI/CD per la build APK
```

**Regola fondamentale:** si modificano solo i file dentro `www/` (HTML, CSV,
immagini). La cartella `android/` non si tocca mai a mano — viene aggiornata
automaticamente dallo step `npx cap sync android` nel workflow.

---

## Setup iniziale (una tantum)

### 1. Crea la cartella di progetto e installa Capacitor

```bash
mkdir esperto-alberi-app
cd esperto-alberi-app
npm init -y
npm install @capacitor/core @capacitor/cli @capacitor/android \
            @capacitor/filesystem @capacitor/share
```

### 2. Inizializza Capacitor

```bash
npx cap init "Esperto Alberi" "it.tuonome.espertoalberi" --web-dir=www
```

Questo crea `capacitor.config.ts`. Subito dopo, **sostituiscilo** con la versione
JSON (più compatibile con ambienti CI/CD senza TypeScript):

```bash
rm capacitor.config.ts
```

Crea `capacitor.config.json`:

```json
{
  "appId": "it.tuonome.espertoalberi",
  "appName": "Esperto Alberi",
  "webDir": "www"
}
```

### 3. Crea la cartella www con i tuoi file

```bash
mkdir -p www/immagini/{foglie,foglie_dettaglio,tronchi,frutti,chiome}
cp /percorso/esperto_alberi.html www/index.html
cp /percorso/alberi.csv www/
# copia le immagini nelle sottocartelle corrispondenti
```

### 4. Aggiungi la piattaforma Android

```bash
npx cap add android
```

### 5. Crea il keystore di debug fisso

Questo è il certificato con cui viene firmato ogni APK. Avere un keystore fisso
(invece di uno generato casualmente ad ogni build) è essenziale per poter
**aggiornare l'app senza doverla disinstallare** ogni volta.

```bash
mkdir -p android/keystore
keytool -genkeypair -v \
  -keystore android/keystore/debug.keystore \
  -alias androiddebugkey \
  -storepass android \
  -keypass android \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -dname "CN=Android Debug,O=Android,C=US"
```

### 6. Configura Gradle per usare il keystore fisso

In `android/app/build.gradle`, trova il blocco `buildTypes` e aggiungici
`signingConfigs` subito prima:

```gradle
    signingConfigs {
        debug {
            storeFile file('../keystore/debug.keystore')
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
    }
    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```

Nota: il percorso è `../keystore/` (con `..`) perché `build.gradle` si trova
dentro `android/app/` e deve risalire di un livello per trovare `android/keystore/`.

### 7. Aggiungi l'indicatore di versione nell'HTML (opzionale ma utile)

Aggiunge una piccola scritta in fondo all'app con il commit e la data della build —
comodo per verificare di star testando sempre la versione giusta.

In fondo a `www/index.html`, appena prima di `</body>`:

```html
    <div style="text-align:center; padding:8px; font-size:11px; color:#999;">
        Build: __BUILD_INFO__
    </div>
```

Il placeholder `__BUILD_INFO__` viene sostituito automaticamente dal workflow
con il commit SHA e la data/ora della build.

### 8. Crea il workflow GitHub Actions

Crea il file `.github/workflows/build-apk.yml`:

```yaml
name: Build APK Android

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: Installazione dipendenze npm
        run: npm ci

      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      - name: Iniezione info di build (commit + data)
        run: |
          INFO="${GITHUB_SHA::7} - $(date -u +'%Y-%m-%d %H:%M UTC')"
          sed -i "s/__BUILD_INFO__/$INFO/" www/index.html

      - name: Sincronizzazione asset web -> progetto Android
        run: npx cap sync android

      - name: Build APK di debug
        working-directory: android
        run: |
          chmod +x gradlew
          ./gradlew assembleDebug --no-daemon

      - name: Carica l'APK come artifact scaricabile
        uses: actions/upload-artifact@v4
        with:
          name: esperto-alberi-debug-apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
          retention-days: 30
```

### 9. Crea il .gitignore

```
# Node
node_modules/
npm-debug.log

# Capacitor (rigenerato automaticamente da "npx cap sync")
android/app/src/main/assets/public/
android/app/src/main/assets/capacitor.config.json

# Android / Gradle
android/build/
android/app/build/
android/.gradle/
android/local.properties
android/.idea/

# Keystore di release (NON committare mai la chiave di firma release!)
*.keystore
*.jks
!debug.keystore

# macOS
.DS_Store
```

### 10. Primo push su GitHub

Crea un repository **vuoto** su [github.com/new](https://github.com/new)
(senza README, senza .gitignore, senza licenza), poi:

```bash
git init
git branch -M main
git remote add origin git@github.com:TUO-USERNAME/esperto-alberi-app.git
git add .
git status          # verifica che ci siano android/, .github/, www/
git commit -m "Setup iniziale app Capacitor"
git push -u origin main
```

---

## Workflow quotidiano

Dopo il setup iniziale, il flusso di lavoro è sempre questo:

```bash
# 1. Modifica i file in www/ (HTML, CSV, immagini)

# 2. Verifica localmente in browser (opzionale ma comodo)
npx serve www

# 3. Commit e push
git add -A
git commit -m "Descrizione della modifica"
git push
```

Il push su `main` fa partire automaticamente la build su GitHub Actions.

**Per scaricare l'APK:**
1. Vai su GitHub → tab **Actions**
2. Apri la run più recente (deve avere ✔ verde)
3. In fondo alla pagina → sezione **Artifacts** → scarica `esperto-alberi-debug-apk`
4. Estrai lo zip: contiene `app-debug.apk`

**Per installare sul telefono:**
- Trasferisci `app-debug.apk` sul telefono (Drive, email, cavo USB...)
- Aprilo sul telefono → conferma "Installa da sorgenti sconosciute" (solo la prima volta)
- Gli aggiornamenti successivi non richiedono disinstallazione (grazie al keystore fisso)

**Per aggiungere immagini direttamente da GitHub** (senza passare dal terminale):
- Naviga nella sottocartella giusta su GitHub (es. `www/immagini/chiome/`)
- **Add file → Upload files** → trascina i file → commit su `main`
- Dopo aver fatto upload da browser, ricordati di fare `git pull` dal terminale
  prima del prossimo push locale, per allineare le due storie

---

## Funzionalità native (Diario di Campo)

Le funzionalità native (accesso a filesystem, condivisione) richiedono i plugin
ufficiali Capacitor, installati con:

```bash
npm install @capacitor/filesystem @capacitor/share
```

Dentro l'HTML, si usano i plugin rilevando se si è in ambiente nativo:

```javascript
async function esportaReport(alberoId) {
    // ... costruzione del testo del report ...

    const nomeFile = `Rilevamento_${albero.id}_${new Date().toISOString().slice(0,10)}.txt`;

    if (window.Capacitor && window.Capacitor.isNativePlatform()) {
        // Dentro l'app Android: usa i plugin nativi
        try {
            const { Filesystem, Share } = window.Capacitor.Plugins;
            const fileScritto = await Filesystem.writeFile({
                path: nomeFile,
                data: testoReport,
                directory: 'CACHE',      // stringa letterale, non enum
                encoding: 'utf8'         // stringa letterale, non enum
            });
            await Share.share({
                title: 'Report Diario di Campo',
                text: `Rilevamento: ${albero.nomeComune}`,
                url: fileScritto.uri,
                dialogTitle: 'Salva o condividi il report'
            });
        } catch (err) {
            if (err.message && err.message.includes("canceled")) return;
            alert('Errore: ' + err.message);
        }
        return;
    }

    // Fallback per browser desktop (sviluppo/test)
    const blob = new Blob([testoReport], { type: "text/plain;charset=utf-8" });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = nomeFile;
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}
```

**Attenzione importante:** i valori degli enum Capacitor (`Directory.Cache`,
`Encoding.UTF8` ecc.) non sono disponibili in un'app HTML pura senza bundler
(webpack/vite). Usare sempre i valori stringa letterali corrispondenti:
`'CACHE'`, `'utf8'`, ecc.

---

## Cambiare l'icona dell'app

Prepara un'immagine PNG quadrata **1024x1024 pixel** e salvala come
`resources/icon.png`, poi:

```bash
npm install @capacitor/assets --save-dev
npx capacitor-assets generate --android
git add android/app/src/main/res/ resources/
git commit -m "Aggiorno icona app"
git push
```

Il plugin genera automaticamente tutte le dimensioni richieste da Android.

---

## Debug da remoto (Chrome DevTools)

Per ispezionare l'app in esecuzione sul telefono:

1. Sul telefono: **Impostazioni → Info sul telefono** → tocca 7 volte **Numero build**
   → si attivano le **Opzioni sviluppatore**
2. **Opzioni sviluppatore → Debug USB** → attiva
3. Collega il telefono al Mac con cavo USB (deve supportare il trasferimento dati,
   non solo la ricarica)
4. Sul telefono, cambia la modalità USB in **Trasferimento file** (MTP)
5. Sul Mac, apri Chrome e vai su `chrome://inspect/#devices`
6. Cerca la voce **"Sistema Esperto Botanico Forestale / https://localhost/"**
   (assicurati che non dica `hidden` — l'app deve essere in primo piano sul telefono)
7. Clicca **inspect** → si apre DevTools collegato in tempo reale alla WebView

Utile per vedere errori JavaScript (tab **Console**) e il codice realmente
in esecuzione (tab **Sources** → cerca il file o la funzione con Cmd+F).

---

## Note importanti

**Usa sempre `capacitor.config.json` e non `.ts`**
La versione TypeScript richiede che TypeScript sia installato anche nel runner CI,
il che può causare errori di build. La versione JSON non ha dipendenze aggiuntive.

**JDK 21 è richiesto da Capacitor 8.x**
Versioni precedenti (es. JDK 17) causano `invalid source release: 21` durante
la compilazione Java. Il workflow usa già JDK 21.

**Non modificare mai la cartella `android/` a mano**
Eccetto `android/app/build.gradle` (per la configurazione della firma) e
`android/keystore/` (per il keystore). Tutto il resto viene sovrascritto
da `npx cap sync android` ad ogni build.

**Il keystore debug va committato nel repository**
È il certificato di firma per gli APK di debug: non è segreto e deve essere
presente nel repository affinché ogni build CI usi lo stesso certificato.
Senza questo, ogni build genera un certificato diverso e l'aggiornamento
dell'app sul telefono richiede la disinstallazione preventiva.

**Quando si mischia terminale e upload da GitHub browser**
Fare un `git pull` prima di ogni push locale, per evitare divergenze tra
le due storie. Impostare una volta per tutte il comportamento di default:
```bash
git config --global pull.rebase false
```

---

## Prossimi passi futuri

- **Pubblicazione sul Play Store**: richiede un APK/AAB firmato con una chiave
  di *release* (diversa dal debug keystore), generata localmente e salvata
  come GitHub Secret (mai nel repository). La chiave di release, una volta
  usata per pubblicare, non può essere cambiata — conservarla con cura.
- **Splash screen personalizzato**: come per le icone, basta aggiungere
  `resources/splash.png` (2732x2732 pixel) e rilanciare
  `npx capacitor-assets generate --android`.
- **Aggiornamenti OTA (Over The Air)**: con Capacitor Live Updates è possibile
  aggiornare la parte web dell'app (HTML/CSS/JS) senza passare per una nuova
  build APK — utile per correzioni rapide senza dover reinstallare.
