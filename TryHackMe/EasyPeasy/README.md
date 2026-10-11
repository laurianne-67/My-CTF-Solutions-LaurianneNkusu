# Write-up : Easy Peasy

- **Plateforme :** TryHackMe
- **Difficulté :** Facile
- **Objectif :** Énumération, trouver les flags cachés, décoder des hashs et escalader les privilèges.

## 1. Reconnaissance (Nmap)
Le scan initial révèle plusieurs ports ouverts, dont un serveur web sur un port non standard.
```bash
nmap -sV -sC <IP_CIBLE>
```

## 2. Énumération Web & Découverte
En explorant le site web et en analysant les fichiers cachés (comme le fichier `robots.txt` ou l'inspection du code source), j'ai pu identifier des répertoires cachés et des indices importants.

## 3. Cryptographie & Décodage
Le challenge contient des messages encodés ou des hashs cachés (par exemple, dans les commentaires ou des répertoires spécifiques). 
- Outils utilisés : CyberChef / Outils de décodage MD5.

## 4. Flags obtenus
- **User Flag :** `[Insérer le flag ici]`
- **Root Flag :** `[Insérer le flag ici]`
