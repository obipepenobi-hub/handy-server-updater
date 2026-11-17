# Anleitung: Handy-server über das Update-Repo ausliefern und in Liams Appstore aufnehmen

Für die Claude-Sitzung, die an Handy-server arbeitet, und für Liam. Stand: 17.11.2026.

## Kurz gesagt

- Neue Versionen von Handy-server erscheinen in diesem **Update-Repo** (`obipepenobi-hub/handy-server-updater`,
  öffentlich, nur Releases). Der Programmcode kann in einem eigenen (auch privaten) Repo liegen.
- Die App prüft hier auf Updates, lädt die APK, vergleicht die SHA-256-Prüfsumme und übergibt sie dem
  System-Installer.
- Jede Version geht zusätzlich in **Liams Appstore** (Ordner `incoming-apks/`). Für das allererste
  Listing braucht der Store einmalig eine neue Store-Version (Signer-Pinning), siehe Abschnitt 6.
- Der Signierschlüssel für die App **existiert schon** (Abschnitt 1). Mit keinem anderen Schlüssel signieren,
  vor allem nicht mit dem Android-Debug-Schlüssel: Der Store und das Release-Skript lehnen das ab.

## 1. Eckdaten

| | |
|---|---|
| App | Handy-server – App zum Hosten von Minecraft-Servern |
| Paketname | Vorschlag `com.liamkkm.handyserver` (passend zu `com.liamkkm.cadence` und `com.liamkkm.mio`). Hat die App schon eine andere `applicationId`, **die bestehende nehmen** und `PACKAGE` in `tools/publish_release.py` vor dem ersten Release anpassen. Danach **nie ändern**. |
| Update-Repo | `obipepenobi-hub/handy-server-updater` (öffentlich, nur Releases) |
| Code-Repo | noch keins; die App-Sitzung legt es an (privat ist möglich, deshalb ist dieses Update-Repo getrennt und öffentlich) |
| Dateien pro Release | `Handy-server.apk`, `SHA256SUMS.txt`, `android-update.json` |
| Version | Datei `VERSION` (x.y.z) · Tag `vX.Y.Z` · `versionCode = major*10000 + minor*100 + patch` |
| Signierschlüssel | `%USERPROFILE%\handy-server-signing\handy-server-release.p12` (PKCS12, RSA 4096, Alias `handyserver`), Passwort in `keystore-password.txt` im selben Ordner – nie ins Git, nie in Chats oder Logs ausgeben |
| Zertifikat (SHA-256) | `845a55a793ec58e4ca1308ed5aeb6302ab94e114a10356110702dbcf01a70cf3` |
| Sicherungen des Schlüssels | `D:\Backups\handy-server-signing\` und `G:\Backups\handy-server-signing\` |
| Lokaler Klon dieses Repos | `F:\claude programmiren\handy-server-updater` |

## 2. Einrichten in der App (einmalig)

**Version aus einer Datei** (ein Ort, den das Release-Skript hochzählt), Beispiel für `app/build.gradle`
(in Kotlin-DSL sinngemäß; Pfad von `VERSION` an das Projekt anpassen):

```groovy
def versionNameStr = new File(rootDir, "VERSION").text.trim()
def vp = versionNameStr.tokenize('.')
def versionCodeInt = (vp[0] as Integer) * 10000 + (vp[1] as Integer) * 100 + (vp[2] as Integer)
android { defaultConfig { versionCode versionCodeInt; versionName versionNameStr } }
```

**Signieren mit dem privaten Schlüssel**, der **außerhalb** des Projekts liegt (so landet er nie im Git):

```groovy
def signingDir = new File(System.getenv("HANDY_SERVER_SIGNING") ?: (System.getProperty("user.home") + "/handy-server-signing"))
def passwordFile = new File(signingDir, "keystore-password.txt")
android {
    signingConfigs {
        release {
            if (passwordFile.exists()) {
                storeFile new File(signingDir, "handy-server-release.p12")
                storeType "pkcs12"
                storePassword passwordFile.text.trim()
                keyAlias "handyserver"
                keyPassword passwordFile.text.trim()
            }
        }
    }
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            if (passwordFile.exists()) signingConfig signingConfigs.release
        }
    }
}
```

Fehlt der Schlüssel, entsteht eine unsignierte APK; das Release-Skript lehnt sie ab.

**Manifest:** Berechtigungen `INTERNET` und `REQUEST_INSTALL_PACKAGES`, `android:allowBackup="false"`,
`android:usesCleartextTraffic="false"` und ein FileProvider für die Update-Datei:

```xml
<provider android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider" android:exported="false"
    android:grantUriPermissions="true">
    <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@xml/file_paths" />
</provider>
```

mit `res/xml/file_paths.xml`: `<paths><cache-path name="updates" path="updates/" /></paths>`

Außerdem in `.gitignore`: `*.p12`, `*.jks`, `keystore/`, `keystore.properties`, `dist/`.

## 3. Anforderungen an den Updater in der App

Als Muster dient der `Updater.kt` aus `obipepenobi-hub/mio2-app` (Pfad
`android/app/src/main/java/com/liamkkm/mio/Updater.kt`), mit diesen Verschärfungen:

1. **Quelle:** `https://github.com/obipepenobi-hub/handy-server-updater/releases/latest/download/android-update.json`
   lesen (kein API-Limit). Inhalt: `version`, `versionCode`, `url`, `size`, `sha256`.
   Antwort 404 (noch kein Release) heißt „kein Update“, kein Fehlerdialog.
2. Nur dann ein Update anbieten, wenn `versionCode` im JSON größer ist als die installierte
   (`packageManager.getPackageInfo(packageName, 0).longVersionCode`).
3. **Download-Adresse prüfen** (sie kommt aus dem Netz, ist also nicht vertrauenswürdig): `https`, Host `github.com`,
   Pfad beginnt mit `/obipepenobi-hub/handy-server-updater/releases/download/`, kein `..`, kein Port, keine
   Zugangsdaten. Nach Weiterleitungen muss der Host `github.com` oder `*.githubusercontent.com` sein.
   Vorlage: `UpdateUrlPolicy.kt` im Repo `obipepenobi-hub/impulskauf-stopper`.
4. In `cacheDir/updates/` speichern (nie auf externen Speicher), alte Dateien vorher löschen, Größenlimit
   (der Store lädt höchstens 400 MB; die App sollte dasselbe oder weniger erlauben), Abbruch bei mehr Bytes als
   angekündigt.
5. **SHA-256 prüfen:** Prüfsumme der heruntergeladenen Datei == `sha256` aus dem JSON, sonst Datei löschen und
   eine verständliche Meldung zeigen.
6. **Vor dem Installer** prüfen, dass Paketname und Signer der heruntergeladenen APK zur installierten App passen
   (`getPackageArchiveInfo(..., GET_SIGNING_CERTIFICATES)`, vor Android 9 `GET_SIGNATURES`); Vorlage:
   `UpdateInstaller.signatureAcceptable` im Impulskauf-Stopper-Repo. Android prüft beim Installieren sowieso, so gibt
   es aber eine klare Meldung statt „App nicht installiert“.
7. Installation über FileProvider + `ACTION_VIEW`; vorher `canRequestPackageInstalls()` prüfen und sonst zu
   „Aus dieser Quelle installieren“ führen.
8. Repo-Besitzer und -Name sind Konstanten im Code, nichts davon aus Netzantworten übernehmen.

## 4. So entsteht ein Release

1. `VERSION` hochzählen und die Release-APK bauen (`assembleRelease`, signiert mit dem Schlüssel aus Abschnitt 1).
2. Im lokalen Klon dieses Repos ausführen:

   ```
   python tools/publish_release.py PFAD/zur/app-release.apk "Was ist neu" --appstore
   ```

   Das Skript prüft die APK, **bevor** etwas veröffentlicht wird: Paketname, Version (x.y.z, versionCode passt),
   nicht debuggable, Signatur gültig **und** exakt der Schlüssel aus Abschnitt 1, Version neuer als das letzte
   Release. Dann legt es `SHA256SUMS.txt` und `android-update.json` an, erstellt das Release `vX.Y.Z` und legt die APK als
   `handy-server-X.Y.Z.apk` in `liams-appstore/incoming-apks/`.
   `--dry-run` prüft nur und veröffentlicht nichts; ohne `--appstore` bleibt der Appstore unberührt.
3. Danach der Appstore-Sitzung sagen: **„Schau im apk-Ordner nach, es gibt ein Update.“**

Beim **ersten** Release: vorher `PACKAGE` in `tools/publish_release.py` auf die echte `applicationId` setzen (siehe
Abschnitt 1), committen und pushen. Stimmt der Paketname nicht, bricht das Skript ab.

Voraussetzungen: Python 3.8+, Android-SDK build-tools, git, `gh auth login` (alles auf Liams PC vorhanden).

## 5. Hinweis zu Minecraft

Die App ist kein offizielles Minecraft-Produkt. Der Hinweis „NOT AN OFFICIAL MINECRAFT PRODUCT. NOT APPROVED BY OR
ASSOCIATED WITH MOJANG OR MICROSOFT.“ gehört in die README (steht schon hier) und in die Beschreibung im Appstore.
Server-Software von Mojang darf nur zusammen mit der EULA genutzt werden; die App muss den Nutzer die EULA bestätigen
lassen, bevor sie einen Server startet.

## 6. In Liams Appstore aufnehmen

Der Store installiert nur Apps, deren Signer **im Store-Code** hinterlegt ist (`security/TrustedSigners.kt`).
Darum braucht Handy-server einmalig einen Eintrag und damit eine neue Store-Version. Das macht die Appstore-Sitzung
(Projekt `LiamsAppstore`), und eine Store-Version veröffentlicht sie nur nach Rückfrage bei Liam:

1. In `TrustedSigners.kt` ergänzen (mit der echten `applicationId`):
   `"com.liamkkm.handyserver" to setOf("845a55a793ec58e4ca1308ed5aeb6302ab94e114a10356110702dbcf01a70cf3")`
2. In `apps.json` ggf. eine Kategorie `spiele` anlegen und die App eintragen. Vorschlag (Texte anpassen, sobald die
   Funktionen feststehen):

   ```json
   {
     "id": "handy-server",
     "packageName": "com.liamkkm.handyserver",
     "name": "Handy-server",
     "author": "dir",
     "category": "spiele",
     "version": "1.0.0", "versionCode": 10000, "sizeBytes": 0,
     "teaser": "Minecraft-Server hosten.",
     "description": "TODO: kurz beschreiben, was die App kann. Kein offizielles Minecraft-Produkt. Nicht von Mojang oder Microsoft genehmigt oder mit Mojang oder Microsoft verbunden.",
     "tags": ["Kein Konto", "Minecraft"],
     "permissions": [
       { "label": "Internet (Server-Download, Spieler, Update-Check)", "manifestName": "INTERNET" },
       { "label": "Updates installieren (mit deiner Bestätigung)", "manifestName": "REQUEST_INSTALL_PACKAGES" }
     ],
     "changelog": [], "reviews": [], "iconUrl": "", "screenshots": [],
     "apkUrl": "https://github.com/obipepenobi-hub/liams-appstore/releases/download/handy-server-v1.0.0/handy-server-v1.0.0.apk",
     "apkSha256": "<aus tools/apk-info.sh>",
     "minSdk": 26, "installs": 0
   }
   ```

   Die Berechtigungsliste muss zur echten Manifest-Datei passen (weitere Berechtigungen wie Benachrichtigungen oder
   Vordergrunddienst ergänzen).
3. Ablauf pro Version steht in der README von Liams Appstore („Eine neue Version einer gelisteten App
   veröffentlichen“): `incoming-apks/` → `bash tools/apk-info.sh <apk>` (der Signer muss „hinterlegt“ sein) →
   Prerelease `handy-server-vX.Y.Z` im Store-Repo → `apps.json` (`version`, `versionCode`, `sizeBytes`, `apkUrl`,
   `apkSha256`) → Datei aus `incoming-apks/` löschen.

## 7. Checkliste

- [ ] `applicationId` festgelegt, `PACKAGE` im Skript angepasst und gepusht
- [ ] Version aus `VERSION`, Signierung, Manifest und FileProvider nach Abschnitt 2 eingerichtet
- [ ] Updater nach Abschnitt 3 gebaut
- [ ] Erste Release-APK mit `--dry-run` geprüft, dann veröffentlicht (Abschnitt 4)
- [ ] Update von Version n auf n+1 auf dem Handy getestet (Update-Popup, Installation)
- [ ] Appstore-Sitzung: Signer in `TrustedSigners.kt`, Store-Version, Katalogeintrag
- [ ] Schlüssel an einem dritten Ort gesichert (USB-Stick/Passwort-Manager), nicht nur auf diesem PC

## 8. Stand

Erledigt: Update-Repo angelegt, Signierschlüssel erzeugt (privat, PKCS12) und doppelt gesichert. Das Release-Skript
ist dasselbe wie im Mio²-Update-Repo und dort ausführlich getestet (Trockenlauf mit echter APK, Fehlerfälle, ein
Release als Entwurf, Appstore-Upload gegen ein Test-Repo); mit der App von Handy-server selbst wurde es noch nicht
ausprobiert, weil es noch keine APK gibt.
Offen: Punkte aus der Checkliste.
