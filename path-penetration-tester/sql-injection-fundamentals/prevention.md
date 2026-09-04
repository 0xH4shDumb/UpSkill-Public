# Prevention des injections SQL

## Pourquoi

L'injection SQL est une vulnerabilite connue depuis plus de 20 ans, et les contre-mesures sont bien documentees. Pourtant, elle reste dans le top 10 OWASP. Comprendre les mecanismes de prevention est essentiel, autant pour securiser les applications que pour evaluer leur robustesse en pentest.

## Comment ca marche

La cause fondamentale de la SQLi est le melange entre donnees utilisateur et code SQL dans la meme chaine. Toutes les contre-mesures visent a separer ces deux elements.

## En pratique

### Requetes parametrees (prepared statements)

C'est la solution la plus efficace. Au lieu de concatener l'entree utilisateur dans la requete, on utilise des placeholders (`?`) et on lie les valeurs separement. Le SGBD traite les valeurs comme des donnees, jamais comme du code.

{% tabs %}
{% tab title="PHP (mysqli)" %}
```php
$query = "SELECT * FROM logins WHERE username=? AND password=?";
$stmt = mysqli_prepare($conn, $query);
mysqli_stmt_bind_param($stmt, 'ss', $username, $password);
mysqli_stmt_execute($stmt);
$result = mysqli_stmt_get_result($stmt);
```
{% endtab %}

{% tab title="PHP (PDO)" %}
```php
$stmt = $pdo->prepare("SELECT * FROM logins WHERE username=:user AND password=:pass");
$stmt->execute(['user' => $username, 'pass' => $password]);
$result = $stmt->fetchAll();
```
{% endtab %}

{% tab title="Python" %}
```python
cursor.execute("SELECT * FROM logins WHERE username=%s AND password=%s", (username, password))
```
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Les requetes parametrees sont la reference. Elles bloquent toute injection SQL, quelle que soit la complexite du payload. C'est la premiere recommandation a faire dans un rapport.
{% endhint %}

### Sanitisation des entrees

Si les requetes parametrees ne peuvent pas etre utilisees (cas rares), echapper les caracteres speciaux dans les entrees utilisateur :

```php
$username = mysqli_real_escape_string($conn, $_POST['username']);
$password = mysqli_real_escape_string($conn, $_POST['password']);
```

La fonction echappe les `'`, `"`, `\` et d'autres caracteres speciaux pour qu'ils soient traites comme du texte.

{% hint style="warning" %}
L'echappement est moins fiable que les requetes parametrees. Certains jeux de caracteres (encodages multi-octets) peuvent contourner l'echappement. Toujours preferer les prepared statements.
{% endhint %}

### Validation des entrees

Valider que l'entree correspond au format attendu avant de l'utiliser :

```php
$pattern = "/^[A-Za-z\s]+$/";
if (!preg_match($pattern, $_GET['port_code'])) {
    die("Entree invalide.");
}
```

Si un champ ne doit contenir que des lettres, rejeter toute entree contenant d'autres caracteres. Cette approche est complementaire aux requetes parametrees.

### Principe de moindre privilege

L'utilisateur de base de donnees utilise par l'application ne doit avoir que les privileges strictement necessaires :

```sql
-- Creer un utilisateur avec des droits minimaux
CREATE USER 'webapp'@'localhost' IDENTIFIED BY 'strong_password';
GRANT SELECT ON appdb.public_data TO 'webapp'@'localhost';
```

Meme si une SQLi est exploitee, l'attaquant est limite aux operations autorisees pour cet utilisateur. Pas de `FILE` privilege = pas de lecture/ecriture de fichiers. Pas de `SUPER` = pas d'operations d'administration.

### Web Application Firewall (WAF)

Les WAF (ModSecurity, Cloudflare, AWS WAF) analysent les requetes HTTP et bloquent celles contenant des patterns de SQLi (`UNION SELECT`, `INFORMATION_SCHEMA`, `OR 1=1`, etc.).

Les WAF sont une couche de defense supplementaire, pas un remplacement des bonnes pratiques de code. Un attaquant determine peut souvent les contourner avec des techniques d'evasion (encodage, casse alteree, commentaires inline).

## Defense en profondeur

La meilleure protection combine plusieurs couches :

| Couche | Mesure |
|---|---|
| Code | Requetes parametrees (obligatoire) |
| Code | Validation des entrees (complementaire) |
| Base de donnees | Principe de moindre privilege |
| Infrastructure | WAF (filtrage des patterns) |
| Monitoring | Logs et alertes sur les erreurs SQL |

## Pieges et galeres

- Les ORM (Eloquent, SQLAlchemy, ActiveRecord) utilisent des requetes parametrees en interne, mais permettent aussi les requetes brutes. Verifier que les requetes brutes ne sont pas utilisees avec des entrees non sanitisees.
- Les stored procedures ne sont pas automatiquement securisees. Si elles construisent des requetes dynamiques avec concatenation, elles sont tout aussi vulnerables.
- La validation cote client (JavaScript) ne protege pas contre la SQLi. L'attaquant peut modifier la requete directement avec Burp Suite ou curl.
- L'echappement avec `addslashes()` en PHP n'est pas suffisant. Utiliser les fonctions specifiques au SGBD (`mysqli_real_escape_string`, `pg_escape_string`).

## Retour terrain

En pentest, quand une SQLi est trouvee, la recommandation principale est toujours la meme : migrer vers des requetes parametrees. C'est la correction definitive. Les WAF et la validation sont des couches supplementaires, pas des substituts.

Dans le rapport, documenter non seulement la vulnerabilite et son exploitation, mais aussi la recommandation concrete avec un exemple de code corrige adapte au langage de l'application.

## Memo express

| Mesure | Efficacite | Priorite |
|---|---|---|
| Requetes parametrees | Bloque toute SQLi | Obligatoire |
| Validation des entrees | Reduit la surface d'attaque | Recommande |
| Echappement | Protege partiellement | Dernier recours |
| Moindre privilege | Limite l'impact | Recommande |
| WAF | Filtre les patterns connus | Complementaire |

***
