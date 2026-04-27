# LAB10-SECURITE
# 🔬 Lab Frida — Analyse dynamique Android avec UnCrackable Level 1

> **Environnement utilisé**
> - OS : Windows (PowerShell)
> - Cible : Émulateur Android AVD (emulator-5556)
> - Application : OWASP UnCrackable Level 1 (`owasp.mstg.uncrackable1`)
> - Frida version : 17.9.1

---

## Étape 1 — Installation du client Frida

Installation de Frida et des outils CLI côté PC.
<img width="898" height="99" alt="python-version" src="https://github.com/user-attachments/assets/050e8b09-f214-42db-86b1-b5600ceb0f16" />
```powershell
pip install --upgrade frida frida-tools
```
<img width="903" height="146" alt="install-frida" src="https://github.com/user-attachments/assets/f0ba23ea-4307-4f4c-ab53-0ee502b8b2a3" />


**Vérification :**
```powershell
frida --version
python -c "import frida; print(frida.__version__)"
```
<img width="894" height="350" alt="frida-version" src="https://github.com/user-attachments/assets/c9c71efa-c994-4a15-b8bb-5eac4d161c4a" />

✅ Résultat : `17.9.1`

---

## Étape 2 — Installation d'ADB (Android Platform Tools)

Téléchargement depuis : https://developer.android.com/tools/releases/platform-tools  
Ajout du dossier `platform-tools` au PATH Windows.

**Vérification :**
```powershell
adb version
adb devices
```
<img width="510" height="146" alt="adb-devices" src="https://github.com/user-attachments/assets/be15e37e-8f22-4e39-aa74-1ee02210d53b" />

---

## Étape 3 — Déploiement de frida-server sur l'émulateur

### 3.1 Identifier l'architecture CPU
```powershell
adb shell getprop ro.product.cpu.abi
# Résultat : x86_64
```

### 3.2 Télécharger frida-server
Depuis : https://github.com/frida/frida/releases  
Fichier téléchargé : `frida-server-17.9.1-android-x86_64.xz`  
Décompression avec **7-Zip** sous Windows.
<img width="532" height="36" alt="frida-server" src="https://github.com/user-attachments/assets/7d1b33e5-9e04-4009-85c0-d2e0aa2332cd" />

### 3.3 Pousser et rendre exécutable
```powershell
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
```
<img width="524" height="122" alt="demarrage-frida-server" src="https://github.com/user-attachments/assets/e50c9d95-8418-49ea-8fa5-f66a10d3a75d" />

### 3.4 Activer root ADB (nécessaire sur émulateur AVD)
```powershell
adb -s emulator-5556 root
# Résultat : restarting adbd as root
```

### 3.5 Lancer frida-server
```powershell
adb -s emulator-5556 shell "/data/local/tmp/frida-server &"
```
<img width="560" height="49" alt="verification-frida-server" src="https://github.com/user-attachments/assets/dc0b6949-7ff0-4a5b-bd4a-79be5038c465" />

### 3.6 Configurer la redirection de ports
```powershell
adb -s emulator-5556 forward tcp:27042 tcp:27042
adb -s emulator-5556 forward tcp:27043 tcp:27043
```
<img width="379" height="86" alt="adb-forward" src="https://github.com/user-attachments/assets/df4c5ee5-c2da-4653-9c53-11164a99186e" />

### 3.7 Vérification
```powershell
adb -s emulator-5556 shell ps | findstr frida
# Résultat : root  2862  ...  frida-server
```

---

## Étape 4 — Test de connexion depuis le PC

```powershell
frida-ps -D emulator-5556 -ai
```

✅ Liste des processus de l'émulateur visible, dont :

<img width="413" height="359" alt="frida-ps-u" src="https://github.com/user-attachments/assets/9186e866-8774-469b-8242-7fb0650a820e" />

<img width="475" height="298" alt="frida-ps-uai" src="https://github.com/user-attachments/assets/8cdf4222-0502-4137-8f2d-92144f8878b0" />

```
3043  Uncrackable Level 1  owasp.mstg.uncrackable1
```

> **Problème rencontré** : Deux émulateurs actifs simultanément (5554 et 5556).  
> **Solution** : Utiliser `-D emulator-5556` au lieu de `-U` pour cibler explicitement le bon émulateur.

---

## Étape 5 — Injection minimale

### 5.1 Test Java — hello.js
<img width="414" height="119" alt="hello js" src="https://github.com/user-attachments/assets/4fda4d73-fb49-4d6a-b979-b05773bba349" />

```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");
});
```

```powershell
frida -D emulator-5556 -f owasp.mstg.uncrackable1 -l hello.js
```
<img width="665" height="217" alt="frida-java-perform" src="https://github.com/user-attachments/assets/b66a72bb-b6ed-432c-acd9-fe283a5d4126" />

✅ Résultat : `[+] Frida Java.perform OK`

### 5.2 Test natif — hello_native.js
```javascript
console.log("[+] Script chargé");

Interceptor.attach(Module.getExportByName(null, "recv"), {
  onEnter(args) {
    console.log("[+] recv appelée");
  }
});
```

```powershell
frida -D emulator-5556 -f owasp.mstg.uncrackable1 -l hello_native.js
```

✅ Résultat : `[+] Script chargé`

### Bypass anti-root — bypass.js

UnCrackable Level 1 détecte le root et appelle `System.exit()`.  
Script de contournement utilisé pour maintenir l'app ouverte :

```javascript
Java.perform(function () {
  console.log("[+] Bypass chargé");

  var System = Java.use("java.lang.System");
  System.exit.implementation = function (code) {
    console.log("[!] System.exit(" + code + ") bloqué");
  };

  var Activity = Java.use("android.app.Activity");
  Activity.finish.implementation = function () {
    console.log("[!] Activity.finish() bloqué");
  };

  console.log("[+] Frida Java.perform OK");
});
```

> **Note** : L'option `-f` (spawn) est indispensable pour injecter avant l'activation de l'anti-debug.  
> L'option `-n` (attach) échoue car le processus est déjà protégé au moment de l'attachement.

---

## Étape 6 — Console interactive Frida

Commandes exécutées directement dans la console Frida :

| Commande | Résultat observé |
|---|---|
| `Process.arch` | `"x64"` |
| `Process.mainModule` | `app_process64` — `/system/bin/app_process64` |
| `Process.id` | PID du processus |
| `Process.platform` | `"linux"` |
| `Process.getModuleByName("libc.so")` | Adresse base, chemin, taille de libc |
| `Process.getModuleByName("libc.so").getExportByName("recv")` | Adresse mémoire de recv |
| `Process.enumerateModules()` | Liste complète des bibliothèques chargées |
| `Process.enumerateThreads()` | Liste des threads actifs |
| `Process.enumerateRanges('r-x')` | Zones mémoire exécutables |
| `Java.available` | `true` |

### Bibliothèques SSL/crypto détectées
```javascript
Process.enumerateModules().filter(m =>
  m.name.indexOf("ssl") !== -1 ||
  m.name.indexOf("crypto") !== -1
)
```
✅ Résultat :
- `libcrypto.so` → `/apex/com.android.conscrypt/lib64/` (1 466 368 bytes)
- `libssl.so` → `/apex/com.android.conscrypt/lib64/` (380 928 bytes)

### Classes Java de l'application
```javascript
Java.perform(function () {
  Java.enumerateLoadedClasses({
    onMatch: function (name) {
      if (name.indexOf("uncrackable") !== -1) console.log(name);
    },
    onComplete: function () { console.log("Fin de l'énumération"); }
  });
});
```
✅ Résultat :
- `sg.vantagepoint.uncrackable1.MainActivity$1`
- `sg.vantagepoint.uncrackable1.MainActivity`
<img width="458" height="271" alt="frida-interactif" src="https://github.com/user-attachments/assets/1b202d70-346b-4f93-8b77-6c9b76b79427" />
<img width="600" height="405" alt="frida-interactif2" src="https://github.com/user-attachments/assets/49c7cfb1-d77d-4ec2-9f3b-b1e59f1c96fc" />
<img width="620" height="257" alt="methodes" src="https://github.com/user-attachments/assets/391ab793-9c57-492f-af1b-f5a6c11612d7" />



---

## Étape 7 — Hooks réseau, fichiers et chiffrement

### hook_connect.js — Observation des connexions réseau
<img width="673" height="295" alt="hook_connect" src="https://github.com/user-attachments/assets/87b03814-d665-4c46-840d-a08cc8df808f" />

```javascript
console.log("[+] Hook connect chargé");
const connectPtr = Process.getModuleByName("libc.so").getExportByName("connect");
Interceptor.attach(connectPtr, {
  onEnter(args) {
    console.log("[+] connect appelée — fd = " + args[0]);
  },
  onLeave(retval) {
    console.log("    retour = " + retval.toInt32());
  }
});
```

### hook_network.js — Observation send/recv
<img width="614" height="226" alt="hook_network" src="https://github.com/user-attachments/assets/02921d29-1616-4040-b729-b1bec8c46203" />

```javascript
console.log("[+] Hooks réseau chargés");
const sendPtr = Process.getModuleByName("libc.so").getExportByName("send");
const recvPtr = Process.getModuleByName("libc.so").getExportByName("recv");

Interceptor.attach(sendPtr, {
  onEnter(args) { console.log("[+] send appelée — len = " + args[2].toInt32()); }
});
Interceptor.attach(recvPtr, {
  onEnter(args) { console.log("[+] recv appelée — len demandé = " + args[2].toInt32()); },
  onLeave(retval) { console.log("    recv retourne = " + retval.toInt32()); }
});
```

### hook_file.js — Observation des accès fichiers
<img width="830" height="395" alt="hook_file" src="https://github.com/user-attachments/assets/b081f809-6ad9-4c2f-a74b-3804303c943b" />

```javascript
console.log("[+] Hook fichiers chargé");
const openPtr = Process.getModuleByName("libc.so").getExportByName("open");
Interceptor.attach(openPtr, {
  onEnter(args) {
    this.path = args[0].readUtf8String();
    console.log("[+] open appelée : " + this.path);
  }
});
```

---

## Étape 8 — Hooks Java (SharedPreferences, SQLite, Debug)
### hook_prefs.js — Lecture SharedPreferences
<img width="686" height="229" alt="hook_prefs" src="https://github.com/user-attachments/assets/75106734-23fb-4a24-bc85-c48758e78eef" />

```javascript
Java.perform(function () {
  var Impl = Java.use("android.app.SharedPreferencesImpl");
  Impl.getString.overload("java.lang.String", "java.lang.String").implementation = function (key, defValue) {
    var result = this.getString(key, defValue);
    console.log("[SharedPreferences][getString] key=" + key + " => " + result);
    return result;
  };
});
```

### hook_sqlite.js — Requêtes SQLite
<img width="607" height="229" alt="hook_sqlite" src="https://github.com/user-attachments/assets/dccf793f-ed8a-4695-a244-3096ae9bd5b6" />

```javascript
Java.perform(function () {
  var SQLiteDatabase = Java.use("android.database.sqlite.SQLiteDatabase");
  SQLiteDatabase.rawQuery.overload("java.lang.String", "[Ljava.lang.String;").implementation = function (sql, args) {
    console.log("[SQLite][rawQuery] " + sql);
    return this.rawQuery(sql, args);
  };
});
```

### hook_debug.js — Détection débogueur
<img width="622" height="248" alt="hook_debug" src="https://github.com/user-attachments/assets/0102f09f-7818-4af3-bf36-0a512559bfde" />

```javascript
Java.perform(function () {
  var Debug = Java.use("android.os.Debug");
  Debug.isDebuggerConnected.implementation = function () {
    var result = this.isDebuggerConnected();
    console.log("[Debug] isDebuggerConnected() => " + result);
    return result;
  };
});
```

### hook_runtime.js — Commandes système
<img width="659" height="242" alt="hook_runtime" src="https://github.com/user-attachments/assets/f349d795-530f-483d-982f-b4873927b1bd" />

```javascript
Java.perform(function () {
  var Runtime = Java.use("java.lang.Runtime");
  Runtime.exec.overload("java.lang.String").implementation = function (cmd) {
    console.log("[Runtime.exec] " + cmd);
    return this.exec(cmd);
  };
});
```

### hook_file_java.js — Chemins de fichiers Java
<img width="655" height="259" alt="hook_file_js" src="https://github.com/user-attachments/assets/3daea391-060f-4912-95ad-f15228d6fcb5" />

```javascript
Java.perform(function () {
  var File = Java.use("java.io.File");
  File.$init.overload("java.lang.String").implementation = function (path) {
    console.log("[File] nouveau chemin : " + path);
    return this.$init(path);
  };
});
```

## Commandes de nettoyage

```powershell
# Arrêter frida-server
adb -s emulator-5556 shell pkill -f frida-server

# Supprimer le binaire
adb -s emulator-5556 shell rm /data/local/tmp/frida-server

# Désinstaller côté PC
pip uninstall frida frida-tools
```
