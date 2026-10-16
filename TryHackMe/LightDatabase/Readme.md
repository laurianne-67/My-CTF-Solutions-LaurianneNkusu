# Capture The Flag — Write-up: Light Database (SQL Injection Bypass)

## Résolution du Challenge & Retours d'Expérience

Ce challenge consistait à exploiter une vulnérabilité d'injection SQL (SQLi) sur une application réseau simulant une base de données appelée **Light** (utilisant **SQLite** en arrière-plan). L'application demandait un `username`, construisait une requête SQL sans nettoyer correctement la saisie, puis l'exécutait. La principale difficulté résidait dans le contournement d'un filtre anti-injection (WAF).

---

## Analyse de la Méthodologie Étape par Étape

### Étape 1 — Repérer la faille
En testant un simple caractère d'injection comme `' OR 1=1--`, l'application a renvoyé une erreur de syntaxe différente des réponses "utilisateur non trouvé" habituelles. C'est le signe classique que l'apostrophe `'` casse la structure de la requête interne : l'application ne filtre pas (ou mal) les entrées.

### Étape 2 — Comprendre le filtre (WAF)
Le serveur bloquait explicitement les caractères `/*`, `--` et `%0b` (espace encodé) avec le message suivant :
> `For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)`

Puisque les commentaires classiques (`--`) permettant de tronquer la requête originale étaient interdits, il a fallu **fermer la requête proprement avec une vraie clause SQL valide** (`WHERE '1'='1`) afin que le moteur de base de données valide la syntaxe globale.

### Étape 3 — Caler le nombre de colonnes
L'utilisation de l'opérateur `UNION SELECT` exige impérativement que le nombre de colonnes à gauche et à droite de l'union soit strictement identique. En procédant par tâtonnement (1 colonne, 2 colonnes...), il a été déterminé que la requête originale n'attendait qu'**une seule colonne** principale à afficher.

### Étape 4 — Gérer la quote de fermeture ajoutée par l'application
L'analyse du premier message d'erreur a révélé que l'application ajoutait automatiquement une quote de fermeture `'` suivie de `LIMIT 30` après notre saisie. Il fallait donc **ne pas fermer soi-même** la dernière quote à la fin du payload, sous peine de créer un déséquilibre de caractères (`''`) et de provoquer un crash syntaxique.

### Étape 5 — Lire le schéma via `sqlite_master`
`sqlite_master` est la table système de SQLite contenant la définition de la structure de la base de données (tables, colonnes, index). En effectuant un `UNION SELECT sql FROM sqlite_master`, il est devenu possible de lire directement l'instruction `CREATE TABLE` de la table ciblée (**`admintable`**), révélant ainsi ses colonnes exactes.

### Étape 6 — Résolution du conflit de colonne grâce à l'alias (`AS data`)
L'apparition de l'erreur `no such column: sqlql` indiquait que le code de l'application en aval tentait de lire ou de référencer le résultat de la requête par un nom de colonne bien spécifique. Sans déclaration explicite, le résultat brut de `group_concat(sql)` entrait en conflit avec la logique applicative. L'utilisation d'un alias de colonne (**`AS data`**) a permis de renommer proprement la sortie et de contourner cette ambiguïté pour afficher les données.

---

##  Résumé du Payload Final Utilisé

```sql
smokey' uNiOn sElEcT group_concat(sql) AS data FROM sqlite_master WHERE '1'='1
```

---

##  Méthodologie Générale pour une SQLi en CTF Network

**1. Confirmer l'injection**
Tester des caractères spécifiques (`'`, `"`, `)`) et analyser si le comportement ou l'erreur retournée diverge d'une entrée normale.

**2. Cartographier le filtre méthodiquement**
Tester les mots-clés et caractères un par un (`--`, `#`, `/*`, `UNION`, `SELECT`, `OR`, `AND`, les espaces) pour identifier précisément les protections en place. Consigner ces blocages pour éviter de répéter les mêmes tests.

**3. Trouver des contournements (Bypass)**
*   **Mots-clés filtrés :** Varier la casse (`uNiOn sElEcT`), doubler les lettres si le filtre applique un nettoyage unique non-récursif (`UNIunionON`).
*   **Espaces filtrés :** Tenter les tabulations (`%09`), les sauts de ligne (`%0a`), les commentaires (`/**/`) ou les parenthèses.
*   **Commentaires filtrés :** Compléter la requête de manière syntaxiquement valide (ex: clauses conditionnelles équilibrées).

**4. Déterminer le nombre de colonnes**
Utiliser `ORDER BY n` (de manière incrémentale) ou injecter `UNION SELECT NULL, NULL, ...` jusqu'à la disparition de l'erreur de dimensionnement.

**5. Identifier les colonnes exploitables**
Remplacer les valeurs `NULL` ou factices une par une par des chaînes de test pour repérer lesquelles sont réellement lues et affichées à l'écran par l'application.

**6. Énumérer le schéma avant d'extraire**
Ne jamais chercher à deviner les noms des tables ou des colonnes à l'aveugle. Interroger systématiquement les tables de métadonnées selon le SGBD :
*   **SQLite :** `sqlite_master` (colonnes via le champ `sql`)
*   **MySQL / PostgreSQL :** `information_schema.tables` et `information_schema.columns`

**7. Extraire les données**
Utiliser des fonctions d'agrégation comme `group_concat()` (SQLite/MySQL) ou `string_agg()` (PostgreSQL) pour regrouper plusieurs lignes de résultats en une seule ligne d'affichage, facilitant la récupération via les connexions de type *Reverse Shell* ou *Netcat*.

**8. Tenir un journal des tests**
Lors de sessions `nc` interactives, l'historique peut vite devenir illisible. Noter chaque structure de payload testée et son résultat associé permet d'accélérer drastiquement le diagnostic et l'isolation des erreurs de développement.
