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

```powershell
pip install --upgrade frida frida-tools
```

**Vérification :**
```powershell
frida --version
python -c "import frida; print(frida.__version__)"
```

✅ Résultat attendu : `17.9.1`

---

## Étape 2 — Installation d'ADB (Android Platform Tools)

Téléchargement depuis : https://developer.android.com/tools/releases/platform-tools  
Ajout du dossier `platform-tools` au PATH Windows.

**Vérification :**
```powershell
adb version
adb devices
```

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

### 3.3 Pousser et rendre exécutable
```powershell
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
```

### 3.4 Activer root ADB (nécessaire sur émulateur AVD)
```powershell
adb -s emulator-5556 root
# Résultat : restarting adbd as root
```

### 3.5 Lancer frida-server
```powershell
adb -s emulator-5556 shell "/data/local/tmp/frida-server &"
```

### 3.6 Configurer la redirection de ports
```powershell
adb -s emulator-5556 forward tcp:27042 tcp:27042
adb -s emulator-5556 forward tcp:27043 tcp:27043
```

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
```
3043  Uncrackable Level 1  owasp.mstg.uncrackable1
```

> **Problème rencontré** : Deux émulateurs actifs simultanément (5554 et 5556).  
> **Solution** : Utiliser `-D emulator-5556` au lieu de `-U` pour cibler explicitement le bon émulateur.

---

## Étape 5 — Injection minimale

### 5.1 Test Java — hello.js
```javascript
Java.perform(function () {
  console.log("[+] Frida Java.perform OK");
});
```

```powershell
frida -D emulator-5556 -f owasp.mstg.uncrackable1 -l hello.js
```

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

---

## Étape 7 — Hooks réseau, fichiers et chiffrement

### hook_connect.js — Observation des connexions réseau
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
```javascript
Java.perform(function () {
  var File = Java.use("java.io.File");
  File.$init.overload("java.lang.String").implementation = function (path) {
    console.log("[File] nouveau chemin : " + path);
    return this.$init(path);
  };
});
```

---

## Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| `Failed to spawn: unable to find application` | Package incorrect (`sg.vantagepoint` au lieu de `owasp.mstg`) | Utiliser `frida-ps -D emulator-5556 -ai` pour trouver le bon identifiant |
| `Failed to attach: process not found` | Deux émulateurs actifs, Frida prenait le mauvais | Utiliser `-D emulator-5556` au lieu de `-U` |
| `unexpectedly timed out trying to sync up with agent` | frida-server planté ou protection ptrace active | Relancer frida-server + utiliser `-f` (spawn) |
| `su: invalid uid/gid '-c'` | Image Google Play, `su -c` non supporté | Utiliser `adb -s emulator-5556 root` à la place |
| `InvocationTargetException` | App crashe avant injection avec `-f` | Utiliser `-n` après lancement manuel, ou bypass + `-f` |
| App se ferme (anti-root) | UnCrackable détecte root et appelle `System.exit()` | Injecter `bypass.js` qui neutralise `System.exit()` |

---

## Commandes de nettoyage

```powershell
# Arrêter frida-server
adb -s emulator-5556 shell pkill -f frida-server

# Supprimer le binaire
adb -s emulator-5556 shell rm /data/local/tmp/frida-server

# Désinstaller côté PC
pip uninstall frida frida-tools
```

---

## Références

- Site officiel Frida : https://frida.re/
- Releases frida-server : https://github.com/frida/frida/releases
- OWASP UnCrackable Apps : https://github.com/OWASP/owasp-mastg/tree/master/Crackmes
- Android Platform Tools : https://developer.android.com/tools/releases/platform-tools
