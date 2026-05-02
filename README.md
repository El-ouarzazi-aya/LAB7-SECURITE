# LAB7-SECURITE
# Lab — Analyse dynamique Android avec MobSF et DIVA

## Objectif

Réaliser une analyse dynamique complète d'une application Android vulnérable (DIVA) à l'aide de MobSF, en interceptant les logs runtime, le trafic réseau, et en instrumentant l'application avec Frida.

---

## Environnement

| Composant | Version / Détail |
|---|---|
| Outil d'analyse | MobSF (Mobile Security Framework) via Docker |
| Application cible | DIVA (Damn Insecure and Vulnerable App) |
| Emulateur | Android Virtual Device — Pixel 4a, API 30, sans Google Play |
| Instrumentation | Frida (injecté automatiquement par MobSF) |

---

## Prérequis

- Android Studio avec SDK Platform-Tools (ADB)
- Docker Desktop
- Git
- 8 Go de RAM minimum, processeur 64-bit

---

## Mise en place

### 1. Créer l'AVD

Dans Android Studio, créer un AVD Pixel 4a avec une image API 29 ou 30, architecture x86_64, sans Google Play. Les images avec Google Play sont incompatibles avec MobSF.

### 2. Compiler DIVA

```bash
git clone https://github.com/payatu/diva-android
cd diva-android
./gradlew assembleDebug
```

L'APK est généré dans `app/build/outputs/apk/debug/app-debug.apk`.

### 3. Cloner MobSF

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
```

### 4. Lancer l'emulateur via le script MobSF

Windows :
```powershell
.\scripts\start_avd.ps1 -AVD_NAME <nom_de_votre_avd>
```

Linux / macOS :
```bash
./scripts/start_avd.sh <nom_de_votre_avd>
```

Vérifier la détection :
```bash
adb devices
# Resultat attendu : emulator-5554   device
```

### 5. Lancer MobSF via Docker

```bash
docker pull opensecurity/mobile-security-framework-mobsf:latest

docker run -it --rm \
  -p 8000:8000 \
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 \
  opensecurity/mobile-security-framework-mobsf:latest
```

Remplacer `emulator-5554` par l'identifiant retourné par `adb devices`.

Interface accessible sur : `http://127.0.0.1:8000`  
Identifiants par défaut : `mobsf / mobsf`

---

## Analyse

### Analyse statique

Uploader `app-debug.apk` via "Upload & Analyze". MobSF décompile l'APK et produit un rapport couvrant les permissions, le manifest, les secrets hardcodés, et les problèmes de sécurité détectés statiquement.

Résultat observé : la clé `"pkey" : "notespin"` détectée dans la section Hardcoded Secrets.

### Analyse dynamique

Cliquer sur "Start Dynamic Analysis" dans le rapport. MobSF installe DIVA sur l'emulateur, lance Frida Server et configure le proxy HTTPS global.

---

## Vulnerabilites testees

### Challenge 1 — Insecure Logging

L'application enregistre les entrées utilisateur dans les logs Android. Les identifiants saisis sont visibles en clair dans le Logcat Stream de MobSF.

### Challenge 2 — Hardcoded Secrets

Le mot de passe `notespin` est codé en dur dans le code source. MobSF le détecte automatiquement lors de l'analyse statique.

### Challenge 3 — Insecure Data Storage (SharedPreferences)

Les données sont stockées en clair dans un fichier XML :
```bash
adb shell run-as jakhar.aseem.diva \
  cat /data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
```

### Challenge 4 — Insecure Data Storage (SQLite)

Les données sont stockées sans chiffrement dans une base SQLite :
```bash
adb shell sqlite3 /data/data/jakhar.aseem.diva/databases/ids2 "select * from myuser;"
```

### Challenge 5 — Insecure Data Storage (stockage externe)

Les données sont écrites sur la carte SD, accessible par toutes les applications :
```bash
adb shell cat /sdcard/jakhar.aseem.diva.uinfo.txt
```

### Challenge 7 — Input Validation (injection SQL)

Le champ de recherche est vulnérable à l'injection SQL. Saisir `' OR '1'='1` retourne tous les enregistrements.

### TLS/SSL Security Tests

MobSF a exécuté quatre tests TLS sur DIVA :

| Test | Resultat |
|---|---|
| TLS Misconfiguration | Passe |
| TLS Pinning / Certificate Transparency | Passe |
| TLS Pinning Bypass | Passe |
| Cleartext Traffic | Passe |

---

## Instrumentation Frida

MobSF injecte automatiquement un bridge Frida dans le processus DIVA. Les scripts suivants ont ete actives :

- API Monitoring
- SSL Pinning Bypass
- Root Detection Bypass
- Debugger Check Bypass
- Clipboard Monitor

Le bouton "Spawn & Inject" injecte les scripts et permet d'observer les appels de methodes Java en temps reel.

---

## Rapport final

Generer le rapport depuis l'interface MobSF via le bouton "Generate Report". Le rapport est disponible en PDF et JSON et regroupe les resultats statiques et dynamiques.

---

## Screenshots lors de la réalisaton du lab
<img width="960" height="470" alt="insecure-logging" src="https://github.com/user-attachments/assets/36f14759-dc60-4c4d-a315-60d74823ca99" />
<img width="960" height="509" alt="docker-installation" src="https://github.com/user-attachments/assets/6dc1a0ba-16cc-4785-8973-2c140f9d1413" />
<img width="960" height="470" alt="diva-static-analysis" src="https://github.com/user-attachments/assets/e50296e8-9be6-4704-bf0b-92ef2829ff4f" />
<img width="957" height="478" alt="diva-dynamic-analysis" src="https://github.com/user-attachments/assets/a3e3af28-8709-4b05-923c-b617e6367090" />
<img width="453" height="267" alt="diva-apk-installation" src="https://github.com/user-attachments/assets/5b1464b0-ff5c-4fe9-bac6-4a2906d37477" />
<img width="960" height="494" alt="device-manager" src="https://github.com/user-attachments/assets/40f16c25-14ea-477f-9940-e5439ef299e4" />
<img width="607" height="343" alt="demarrage-avd" src="https://github.com/user-attachments/assets/a4a1840a-5298-443f-a00c-97bc2cac789b" />
<img width="427" height="174" alt="adb-devices" src="https://github.com/user-attachments/assets/3df7095b-b723-470f-9673-dc8bb9ac5c8a" />
<img width="585" height="292" alt="pull-docker-mobsf" src="https://github.com/user-attachments/assets/fadb086a-e84d-44c7-83e9-c1ddb421702e" />
<img width="960" height="414" alt="network-test" src="https://github.com/user-attachments/assets/36b7313e-f81a-4e1d-8606-f16570ccd4f7" />
<img width="544" height="127" alt="mobsf-installation" src="https://github.com/user-attachments/assets/7fbf3645-86ae-41a2-90ed-245510896766" />
<img width="958" height="488" alt="mobdf-login" src="https://github.com/user-attachments/assets/cdb524c6-e5fa-443d-b4bd-2aeeed086f35" />
<img width="960" height="425" alt="logcat-stream" src="https://github.com/user-attachments/assets/7875a129-cf98-41b2-b63e-1f14335f1b2e" />
