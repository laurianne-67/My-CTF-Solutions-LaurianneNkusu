# Write-up : Brooklyn Nine-Nine 

- **Plateforme :** TryHackMe
- **Difficulté :** Facile
- **Concepts :** Énumération de ports, Découverte de note d'identifiants, Escalade via Sudo (Nano - GTFOBins).

## 1. Reconnaissance (Legion)
Le scan avec Legion a révélé les ports suivants ouverts sur la machine cible :
- Port 21 (FTP)
- Port 22 (SSH)
- Port 80 (HTTP)

## 2. Accès SSH
L'analyse et la découverte d'une note de rappel ont révélé les identifiants pour le compte de l'utilisateur `holt` , la stéganographie n'est pas mal :
- **Utilisateur :** `------`
- **Mot de passe :** `------------`

Connexion établie avec succès depuis le terminal :
```bash
ssh ----@<IP_CIBLE>
```

## 3. Escalade de Privilèges
L'exécution de la commande de vérification des droits `sudo -l` a montré que l'utilisateur `---` peut exécuter `/bin/nano` en tant que root sans mot de passe (**NOPASSWD**).

En exploitant une technique de détournement **GTFOBins**, j'ai lancé Nano en mode privilège :
```bash
sudo /bin/nano
```
Puis, en utilisant les raccourcis internes d'appel système de l'éditeur (`Ctrl+R` suivi de `Ctrl+X`), j'ai exécuté la commande suivante pour faire apparaître le shell :
```bash
reset; sh 1>&0 2>&0
```
Résultat : Accès **root** obtenu avec succès ! 
Pour conclure j'ai su comment utilisé legion ainsi que GTFObins 
