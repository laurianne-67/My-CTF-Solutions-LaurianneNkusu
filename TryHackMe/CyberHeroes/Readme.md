# Capture The Flag - CyberHéros: Web Authentication

## Résolution du Challenge & Retours d'Expérience

Dans ce challenge, l'objectif était de contourner une mire de connexion afin de s'authentifier et de récupérer le flag. Cette épreuve a été une excellente leçon sur l'importance de l'énumération et de la compréhension du fonctionnement d'une application avant de lancer des outils automatisés.

---

## Analyse de la Méthodologie

### 1. Pourquoi Burp Suite et Hydra n'étaient pas les bons outils ici
Au départ, la démarche logique face à un formulaire de connexion était de tenter une attaque par force brute (via **Burp Intruder** ou **Hydra**). Cependant, cette approche a rapidement échoué :
* **L'absence de requêtes réseau exploitables :** L'envoi du formulaire déclenchait un pop-up d'erreur instantané sans générer de trafic HTTP interceptable ou de requête `POST` vers le serveur pour traiter la donnée.
* **Le piège de l'automatisation aveugle :** Lancer des outils comme Hydra requiert impérativement de connaître le point d'entrée (`endpoint`), la méthode HTTP (ex: `POST`) et la structure exacte des paramètres envoyés. Sans cela, le brute force tourne dans le vide.

### 2. Le déclic : L'analyse du comportement du Pop-up (JavaScript Client-Side)
L'apparition d'une alerte native du navigateur (`10.113.160.173 says: Incorrect Password...`) a mis en évidence que la validation ne se faisait pas côté serveur, mais **côté client**, directement dans le navigateur en JavaScript.

En inspectant le code source de la page de connexion (`Ctrl + U`), la fonction de vérification a été immédiatement localisée :

```javascript
function authenticate() {
    a = document.getElementById('uname')
    b = document.getElementById('pass')
    const RevereString = str => [...str].reverse().join('');
    if (a.value=="-----" && b.value==RevereString("------")) {
        // ... requête masquée menant au flag
    } else {
        alert("Incorrect Password, try again.. you got this hacker !")
    }
}
```

### 3. Reverse de la logique et extraction du Flag
Le code source nous a livré toutes les réponses sans nécessiter la moindre requête brute force :
* **Username en clair :** `h3ck3rBoi`
* **Password obfusqué :** La fonction `RevereString` appliquait simplement une inversion complète de la chaîne de caractères. 

En inversant manuellement la chaîne cible `"-------"`, on obtient le mot de passe valide : **`------`**.

L'introduction de ces identifiants a déclenché l'affichage direct du flag.



##  Leçons à Retenir (Takeaways)

1. **Lire avant d'attaquer :** Toujours analyser le code source HTML/JS et l'historique réseau (`F12`) avant de sortir les outils lourds. Un mécanisme de sécurité basique ou mal conçu (Client-Side Authentication) se résout souvent en quelques secondes d'analyse visuelle.
2. **Comprendre les messages d'erreur :** Les alertes et pop-ups sont des indicateurs précieux sur l'endroit où la logique est exécutée (Côté client vs Côté serveur).
3. **Être prudent avec le Brute Force :** Le brute force doit être une solution de dernier recours, ciblée et configurée avec précision sur des requêtes applicatives réelles.
