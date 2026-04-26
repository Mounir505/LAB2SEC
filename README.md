# Rapport de Laboratoire : Rooting et Sécurité Android
## Section 1 : Introduction et État de l'Art

### 1.1 Définition du Rooting
Le rooting consiste à obtenir les privilèges **super-utilisateur (root)** sur le système d'exploitation Android. Sur cet environnement basé sur Linux, l'utilisateur root possède l'UID 0, ce qui permet de contourner les restrictions de sécurité standard (Sandboxing) et d'accéder à l'intégralité du système de fichiers. En laboratoire, cette manipulation est essentielle pour analyser le comportement de bas niveau des applications.

### 1.2 Périmètre du Test
* **Application :** [NOM_DE_TON_APP] (version debug)
* **Support :** Émulateur Android (AVD) - Pixel 6 Pro API 31
* **Objectif :** Comprendre les impacts du rooting sur les mécanismes d'intégrité (Verified Boot).
* **Données :** Utilisation exclusive de données fictives.
* **Réseau :** Environnement isolé sans accès aux comptes personnels.

### 1.3 État de la Sécurité (Vérifications Initiales)
Avant toute modification, l'état de l'appareil a été audité via les commandes suivantes :

| Commande | Objectif | Résultat attendu |
| :--- | :--- | :--- |
| `adb shell id` | Vérifier l'identité de l'utilisateur | `uid=2000(shell)` (standard) |
| `getprop ro.boot.verifiedbootstate` | Vérifier l'intégrité au démarrage | `green` (système intègre) |
| `getprop ro.boot.veritymode` | Vérifier l'état de dm-verity | `enforcing` (lecture seule) |


### 1.4 Initialisation de l'environnement et de l'émulateur

La capture d'écran ci-dessous illustre le processus de démarrage de l'environnement de test et la configuration du système de fichiers en mode écriture.

#### Commandes exécutées :
1. **Lister les AVD disponibles :**
   ```powershell
   emulator -list-avds
   ```
   Résultat : Confirmation de la présence du profil Pixel_6_Pro.

2. **Vérification de l'utilisateur initial :**

```powershell
adb shell id
```
Observation : L'utilisateur est identifié comme uid=2000(shell), confirmant que nous démarrons en mode utilisateur restreint avant l'élévation.

3. **Lancement de l'émulateur avec système de fichiers modifiable :**

```PowerShell
emulator -avd Pixel_6_Pro -writable-system
```
***Analyse technique :***
Le flag -writable-system est crucial pour ce laboratoire. Comme indiqué par l'avertissement jaune dans la console (System image is writable), cette commande permet de lever la protection en lecture seule de la partition système pour permettre l'injection d'outils d'audit.

   <img width="2876" height="1622" alt="Screenshot 2026-04-26 183228" src="https://github.com/user-attachments/assets/d6ea2c72-f759-4718-84bc-4bb7cb0dfdae" />


---
## Section 2 : Installation et Préparation de la Cible (UnCrackable Level 1)

### 2.1 Téléchargement et Installation
L'application cible utilisée est **UnCrackable Mobile Dashboard Level 1** de l'OWASP (système de crack-me). L'installation a été réalisée via le pont de débogage Android (ADB).

**Commande d'installation :**
```powershell
adb install UnCrackable-Level1.apk
```
### 1.5 Preuves visuelles de l'environnement de test

| Capture du Terminal (Commandes ADB) | Interface de l'Émulateur (Résultat) |
| :--- | :--- |
| <img src="https://github.com/user-attachments/assets/a56b624a-faf1-4d1d-8814-b4346e6b4651" width="100%" /> | <img src="https://github.com/user-attachments/assets/83e59994-5e0b-41cf-b1e3-0b8b89bdf6fc" width="100%" /> |
| **Description :** installation de uncrackable1 | **Description :** lancement depuis le AVD |

---
## Section 3 : Analyse des Risques et Mesures Défensives

### 3.1 Matrice des Risques (Étape 11)
L'utilisation de l'application **UnCrackable Level 1** dans un environnement rooté expose le système aux risques suivants :

| Risque | Niveau | Description |
| :--- | :--- | :--- |
| **Accès au stockage privé** | Critique | Les données stockées dans `/data/data/owasp.mstg.uncrackable1/` deviennent lisibles (clés, préférences). |
| **Hooking de méthodes** | Élevé | Possibilité de modifier le comportement du code en temps réel avec des outils comme Frida. |
| **Fuite via Logcat** | Moyen | Les messages de debug sensibles peuvent être interceptés par n'importe quel processus root. |
| **Contournement d'intégrité** | Élevé | Modification possible des fichiers binaires ou des bibliothèques (.so) de l'application. |

### 3.2 Mesures de remédiation et Défense (Étape 12)
Pour limiter l'impact d'un environnement compromis, les mesures suivantes sont analysées :

1.  **Détection de Root (Implémentée) :** L'application vérifie la présence de binaires (`su`, `busybox`) et de dossiers spécifiques au root. 
2.  **Détection de Debugger :** Empêcher l'attachement d'un debugger Java (JDWP) pour éviter l'analyse dynamique.
3.  **Vérification de Signature :** L'application peut vérifier au runtime si sa propre signature numérique a été modifiée (anti-tampering).
4.  **Chiffrement des chaînes :** Masquer les secrets (clés de chiffrement) dans le code source pour ralentir l'analyse statique.

### 3.3 Observation de la sécurité active
Lors de l'ouverture de l'application, la mesure de défense n°1 a été déclenchée immédiatement. L'application a identifié les modifications apportées à l'AVD (Section 2) et a refusé de démarrer par mesure de sécurité.

---
## Section 4 : Audit de Sécurité (OWASP MASVS) et Conclusion

### 4.1 Conformité au standard OWASP MASVS (Étapes 13 & 14)
L'audit de l'application a été réalisé en se référant au standard **MASVS (Mobile Application Security Verification Standard)**. 

* **Catégorie ciblée :** MASVS-STORAGE (Sécurité du stockage des données).
* **Objectif (MSTG-STORAGE-1) :** Vérifier que l'application ne stocke pas de données sensibles en clair dans son répertoire privé.
* **Méthodologie :** En utilisant les privilèges root obtenus (`uid=0`), nous avons inspecté le dossier `/data/data/[PACKAGE_NAME]/`.
* **Constat :** L'accès root permet de lire les fichiers `shared_prefs` (fichiers XML) et les bases de données SQLite qui seraient normalement protégés par le sandboxing Android.

### 4.2 Synthèse des Diagnostics (Étape 16)
Les preuves de l'état du système lors de l'audit sont résumées ci-dessous :

| Commande | État constaté | Impact sur l'audit |
| :--- | :--- | :--- |
| `adb shell id` | `uid=0(root)` | Contrôle total du système de fichiers. |
| `verifiedbootstate` | `orange/vide` | Intégrité modifiée pour permettre le test. |
| `su -c id` | Succès | Possibilité d'exécution de scripts privilégiés. |

### 4.3 Clôture du Laboratoire (Étapes 17 à 20)
Afin de garantir l'intégrité de l'environnement pour les tests futurs et de respecter les bonnes pratiques de sécurité, les actions suivantes ont été réalisées :

1.  **Nettoyage des données :** Exécution de la commande `adb shell pm clear [PACKAGE_NAME]` pour supprimer les résidus de l'application.
2.  **Remise à zéro (Wipe Data) :** L'émulateur a été réinitialisé via l'option "Wipe Data" du gestionnaire d'AVD.
3.  **Vérification finale :** Le redémarrage de l'appareil sur l'assistant de configuration initial confirme la suppression effective de toutes les données de l'audit.

**Conclusion :** Ce laboratoire a permis de démontrer qu'un attaquant disposant d'un accès root peut neutraliser la majorité des protections logicielles standards d'une application Android. La défense en profondeur (obfuscation, chiffrement local, détection de root) est donc indispensable.
