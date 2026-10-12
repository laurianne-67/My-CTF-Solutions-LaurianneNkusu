# Ce que j'ai compris sur Burp Suite (Et comment j'ai bypassé mon premier CTF Web)

En étudiant la cybersécurité, on apprend vite que la théorie c'est bien, maîtriser les outils sur le terrain c'est mieux. En me penchant sur le lab *Endgame Trainer / Fool's Mate* sur TryHackMe, j'ai dû utiliser **Burp Suite** pour la première fois pour contourner une sécurité. 

Voici ma fiche récapitulative sur l'importance de cet outil, comment il fonctionne et comment je l'ai utilisé pour plier le challenge.

---

## 💡 Mon analogie : L'histoire du garde et du facteur malin

Pour capter le rôle de Burp Suite, j'aime bien imaginer cette scène :
* **Le Navigateur (Moi) :** J'écris une lettre pour demander une action au serveur et je la mets dans une enveloppe.
* **Le Code JavaScript du site (Le garde) :** Il est debout juste à côté de moi. Il surveille ce que j'écris *avant* que je ferme l'enveloppe. Si j'écris un truc qu'il n'aime pas (comme un coup d'échecs gagnant interdit par le scénario), il m'arrache la feuille des mains et m'empêche d'envoyer la lettre en me sortant une alerte pop-up.
* **Le Serveur Web (Le destinataire) :** Il attend la lettre pour l'exécuter, mais il fait aveuglément confiance au garde. Il se dit : *"Si la lettre arrive jusqu'à moi, c'est que le garde l'a validée, donc le contenu est forcément clean"*. **Grave erreur.**

**C'est là que Burp Suite intervient en mode "Facteur malin" :**
1. J'écris une lettre totalement banale et autorisée pour ne pas énerver le garde (par exemple, je bouge mon Roi sur une case inutile).
2. Le garde regarde la lettre, sourit, valide et me laisse fermer l'enveloppe pour la donner au facteur.
3. Sauf que le facteur, c'est **Burp Suite (Proxy)**. Dès qu'il a l'enveloppe en main, il se cache derrière le mur, l'ouvre, rature mon coup de Roi inoffensif, écrit à la place mon coup de Tour gagnant (celui qui fait échec et mat), referme le tout proprement et va la livrer directement au serveur.
4. Le serveur reçoit la lettre, voit le coup gagnant, valide l'action et me balance le Flag. Le script JavaScript du site n'a rien vu passer !

---

## Le cas pratique : Résolution du CTF d'Échecs

Sur le papier, le script JS du site m'interdisait de déplacer ma tour blanche de `a1` à `a8` pour faire échec et mat, en m'affichant une pop-up de menace pour me bloquer. 

Voici les étapes exactes que j'ai suivies dans ma VM Kali Linux pour contourner ça avec Burp Suite :

### 1. Interception
J'ai ouvert le navigateur Chromium intégré à Burp Suite pour que tout mon trafic passe par lui, puis j'ai activé l'option **`Intercept is ON`** dans l'onglet Proxy.

### 2. Capture du coup légitime
Pour ne pas déclencher le blocage du site, j'ai joué un coup tout à fait autorisé : j'ai déplacé mon Roi blanc de `g1` vers `h2`. Le navigateur s'est figé. Normal, Burp venait de bloquer la requête HTTP `POST /api/move` au vol.

### 3. Falsification des données (Le Tampering)
Dans la fenêtre de Burp, j'ai analysé la requête interceptée. Tout en bas, le site envoyait les données du mouvement au format JSON :
```json
// Ce que le garde JS a validé
{
  "from": "g1",
  "to": "h2"
}
```

J'ai cliqué directement dans le texte de Burp Suite et j'ai modifié les valeurs à la main pour forcer le coup de ma Tour que le site me refusait :
```json
// Ce que j'ai injecté en douce
{
  "from": "",
  "to": ""
} juste un coup de maitre aux échecs
```

### 4. Envoi et Flag
J'ai cliqué sur le bouton orange **`Forward`** (Transmettre). Burp a envoyé ma triche directement à l'API du serveur. Le serveur a traité la demande modifiée sans rechigner, a validé le mat, et le Flag s'est affiché direct sur mon écran.

---

## Ce que j'en retiens (Leçon de sécurité)

Ce lab m'a fait comprendre une règle d'or en sécurité web : **On ne fait JAMAIS confiance aux vérifications côté client (navigateur).** 

Si un développeur se contente de bloquer une action en JavaScript sans mettre une double vérification stricte derrière sur son serveur (côté back-end), son application est totalement vulnérable. Il suffit d'un outil comme Burp Suite pour intercepter la requête, réécrire les règles du jeu, et bypasser la sécurité en 30 secondes chrono. 

C'est la base de la manipulation de requêtes, et c'est super puissant pour tester la robustesse d'une API !
