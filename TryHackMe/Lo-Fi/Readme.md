# Guide d'Exploitation : La Vulnérabilité LFI (Local File Inclusion)

Ce document explique les principes de l'inclusion de fichiers locaux (**LFI**) et détaille la méthodologie pas à pas utilisée pour résoudre le défi **Lo-Fi Music** sur TryHackMe.

---

## 1. La Phase Cruciale : L'Énumération et la Reconnaissance

Dans un CTF de cybersécurité (comme dans un test d'intrusion réel), on ne fonce jamais à l'aveugle. Il faut toujours respecter une méthodologie structurée.

### L'Énumération Réseau
Avant de toucher au site web, la première étape consiste à analyser la machine cible pour découvrir ses ports ouverts et ses services actifs.
* **Outil type :** `nmap -sV <IP_CIBLE>`
* **Objectif :** Vérifier que le port `80` (HTTP) ou `443` (HTTPS) est ouvert, indiquant la présence d'un serveur web.

### L'Énumération des Dossiers et Fichiers cachés
Une fois le serveur web repéré, on cherche à cartographier le site pour découvrir des pages cachées, des fichiers de configuration ou des scripts oubliés.
* **Outils types :** `Gobuster`, `Dirbuster` ou `Ffuf`.
* **Objectif :** Trouver des dossiers d'administration ou des fichiers comme `index.php`, `config.php`, etc.

### L'Inspection du Code Source Web
C'est l'étape clé où l'on inspecte manuellement l'application en quête d'indices :
* Examiner les formulaires, les boutons et les URLs.
* Utiliser la fonctionnalité `view-source:` du navigateur pour lire le code HTML/PHP non interprété.
* **Objectif :** Repérer comment le site fonctionne en arrière-plan et identifier une surface d'attaque.

---

## 2. Qu'est-ce que le LFI (Local File Inclusion) ?

Une vulnérabilité **LFI** survient lorsqu'une application web accepte une entrée utilisateur (souvent via un paramètre dans l'URL) et l'utilise pour charger ou inclure un fichier local sur le serveur, **sans vérifier ni nettoyer cette entrée**.

Si le développeur a écrit un code permissif du type :
```php
include($_GET['page']);
```
L'attaquant peut détourner ce comportement pour forcer le serveur à lire des fichiers système sensibles auxquels il ne devrait pas avoir accès (comme `/etc/passwd` sous Linux).

---

## 3. Comment "Remonter" un Dossier ?

Pour exploiter un LFI, il faut comprendre l'arborescence d'un système de fichiers Linux. 

Par défaut, un site web est souvent stocké profondément dans la machine, par exemple dans le chemin : `/var/www/html/`.
Le drapeau (flag) recherché se trouve quant à lui à la **racine du système de fichiers**, représentée par le symbole **`/`**.

### La syntaxe de remontée : `..`
* En informatique, le symbole **`..`** signifie : *"Sortir du dossier actuel pour aller dans le dossier parent (juste au-dessus)"*.
* La barre oblique **`/`** sert de séparateur entre les répertoires.

### Schéma de la remontée en cascade :
Si le script s'exécute dans `/var/www/html/` :
1. `../` nous fait remonter dans `/var/www/`
2. `../../` nous fait remonter dans `/var/`
3. `../../../` nous fait remonter à la racine **`/`**

*Astuce de hacker :* Si on remonte "trop" haut (par exemple en mettant 6 ou 7 fois `../`), le système Linux bloque sagement à la racine `/` sans générer d'erreur. Mettre une longue chaîne comme `../../../../../../` garantit d'atteindre le sommet de l'arborescence à coup sûr.

---

## 4. Application Pratique : Le CTF Lo-Fi Music

Voici le cheminement exact qui nous a permis de progresser dans le défi :

### Étape 1 : Identification du Paramètre
En inspectant l'URL et le code source du site, nous avons cherché le paramètre dynamique chargé de faire basculer les pages ou les musiques. Nous avons ainsi isolé le paramètre vulnérable :
**`?page=`**

### Étape 2 : Construction de la Charge Utile (Payload)
L'énoncé stipulant que le flag se trouve à la racine du système, et sachant que le fichier s'appelle généralement `flag.txt`, nous avons combiné la technique de remontée de dossier avec le nom de notre cible :
**`../../../../flag.txt`**

### Étape 3 : Injection dans l'URL
Nous avons nettoyé l'URL du navigateur en retirant le mode `view-source:` et nous avons injecté notre payload directement après le paramètre repéré.

En envoyant cette requête, le serveur web exécute l'ordre de remonter l'arborescence jusqu'à la racine, ouvre le fichier `flag.txt` et affiche son contenu textuel directement sur l'écran de l'attaquant. Mission accomplie !
En outre pour aller vous pouvez aussi exploiter cette vulnérabilité pour essayer de tenter un reverse shell uniquement si vous arrivez  à transformer le LFI en RCE (Remote Code Execution)
