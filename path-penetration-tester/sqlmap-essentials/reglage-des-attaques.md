# Reglage des attaques

Par defaut, SQLMap envoie un jeu de payloads relativement restreint. C'est un choix de design : couvrir les cas les plus courants sans noyer la cible sous des milliers de requetes. Mais certaines injections ne tombent pas dans ces cas courants, et il faut alors ajuster le niveau de test, preciser le contexte SQL, ou cibler un type d'injection specifique. C'est la difference entre un scan qui rate une vuln et un scan qui la trouve.

## Pourquoi affiner le reglage

SQLMap organise ses payloads selon deux axes : le **level** (de 1 a 5) et le **risk** (de 1 a 3). Le level controle la profondeur des tests (quels parametres, quels headers), tandis que le risk controle l'agressivite des payloads (certains peuvent modifier des donnees en base).

A level 1 / risk 1 (valeurs par defaut), SQLMap envoie environ 72 payloads. A level 5 / risk 3, on monte a pres de 7 865 payloads. La difference est considerable, autant en termes de couverture que de temps d'execution.

{% hint style="info" %}
La plupart des injections classiques tombent avec les reglages par defaut. N'augmente le level/risk que si le scan initial ne trouve rien alors que tu suspectes une vulnerabilite.
{% endhint %}

## Comment ca marche

### Level (1-5)

Le level determine l'etendue des tests :

| Level | Comportement |
|-------|-------------|
| 1 | Teste les parametres GET/POST standards |
| 2 | Ajoute les tests sur les cookies HTTP |
| 3 | Ajoute les tests sur le header `User-Agent` et `Referer` |
| 4 | Tests supplementaires, payloads plus complexes |
| 5 | Tests exhaustifs sur tous les headers HTTP |

### Risk (1-3)

Le risk determine l'agressivite des payloads :

| Risk | Comportement |
|------|-------------|
| 1 | Payloads inoffensifs uniquement (defaut) |
| 2 | Ajoute des payloads bases sur les requetes time-based lourdes |
| 3 | Ajoute des payloads `OR` (risque de modification/suppression de donnees) |

{% hint style="danger" %}
Un risk de 3 utilise des conditions `OR` dans les clauses `WHERE`. Sur une table sensible, ca peut alterer ou supprimer des donnees. A n'utiliser qu'en environnement de test ou avec l'accord explicite du client.
{% endhint %}

### Prefix et suffix

Certaines injections se trouvent dans un contexte SQL particulier ou le payload brut ne fonctionne pas. Par exemple, si l'application encapsule la valeur dans une clause `LIKE` entre parentheses :

```sql
SELECT * FROM users WHERE name LIKE ('%$input%')
```

Ici, le payload doit d'abord fermer le `%')` avant d'injecter du SQL valide, puis rouvrir un contexte coherent apres. On utilise `--prefix` et `--suffix` pour encadrer chaque payload :

```bash
sqlmap -u "http://<IP_CIBLE>/page.php?name=test" --prefix="%')" --suffix="AND ('1%'='1" --batch
```

SQLMap va construire chaque payload sous la forme : `%') <PAYLOAD> AND ('1%'='1`, ce qui maintient la syntaxe SQL valide autour de l'injection.

### Selection de technique

Par defaut, SQLMap teste les six types d'injection (BEUSTQ). On peut restreindre a un sous-ensemble avec `--technique` :

| Lettre | Type d'injection |
|--------|-----------------|
| B | Boolean-based blind |
| E | Error-based |
| U | UNION query-based |
| S | Stacked queries |
| T | Time-based blind |
| Q | Inline queries |

```bash
# - Tester uniquement Boolean, Error et UNION
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --technique=BEU --batch
```

{% hint style="success" %}
Restreindre les techniques accelere considerablement le scan. Si tu sais deja que la cible renvoie des erreurs SQL, `--technique=E` ira droit au but.
{% endhint %}

### Reglage fin du UNION

L'injection UNION necessite de deviner le nombre de colonnes de la requete originale. SQLMap teste par defaut de 1 a 10 colonnes, mais certaines requetes en ont davantage :

```bash
# - Etendre la plage de colonnes testees
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --union-cols=12-20 --batch
```

Deux autres options utiles :

- `--union-char='a'` : remplace la valeur `NULL` par un caractere specifique dans les colonnes UNION. Certains SGBD ou configurations rejettent les `NULL`.
- `--union-from=users` : specifie une table source pour le `FROM` dans la requete UNION. Necessaire sur certains SGBD comme Oracle qui exigent un `FROM` valide (typiquement `FROM dual`).

### Options de detection avancee

Quand l'application ne renvoie pas de reponses clairement differenciables, ces options aident SQLMap a determiner si l'injection fonctionne :

| Option | Role |
|--------|------|
| `--string="texte"` | Indique une chaine presente dans la reponse quand la condition est vraie |
| `--not-string="texte"` | Chaine presente uniquement quand la condition est fausse |
| `--code=200` | Code HTTP attendu pour une reponse "vraie" |
| `--titles` | Compare les titres de page plutot que le contenu complet |
| `--text-only` | Ignore le HTML, compare uniquement le texte visible |

```bash
# - Utiliser le titre de page comme critere de distinction
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --titles --batch
```

{% hint style="info" %}
`--string` est particulierement utile quand SQLMap hesite sur la validite de l'injection. Donne-lui un mot ou une phrase qui n'apparait que dans la reponse "normale" (par exemple le nom d'un utilisateur affiche quand l'ID est correct).
{% endhint %}

## En pratique

Scenario typique : le scan par defaut ne detecte rien, mais tu suspectes une injection sur un parametre POST envoye via un formulaire complexe.

```bash
# - Premiere passe : augmenter level et risk
sqlmap -u "http://<IP_CIBLE>/search.php" --data="query=test&cat=1" -p "query" --level=3 --risk=2 --batch
```

Si ca ne donne toujours rien, on peut combiner avec un prefix/suffix adapte au contexte :

```bash
# - Ajouter prefix/suffix pour un contexte SQL specifique
sqlmap -u "http://<IP_CIBLE>/search.php" --data="query=test&cat=1" -p "query" --level=5 --risk=3 --prefix="')" --suffix="-- -" --batch
```

Et pour accelerer un scan deja cible :

```bash
# - Restreindre aux techniques pertinentes avec plage UNION etendue
sqlmap -u "http://<IP_CIBLE>/page.php?id=1" --technique=EU --union-cols=5-15 --union-char='a' --batch
```

## Pieges et galeres

**Augmenter le level/risk sans raison** : passer directement a level 5 / risk 3 multiplie le temps de scan par 100 et genere un trafic enorme. Commence toujours par les valeurs par defaut, puis monte progressivement.

**Oublier le contexte SQL** : si l'injection se trouve dans une sous-requete, un `LIKE`, ou une fonction, le payload brut ne fonctionnera jamais. Analyse la requete cote application (ou devine-la a partir du comportement) avant de lancer SQLMap.

**Risk 3 en production** : les payloads `OR` peuvent vider une table entiere si la clause `WHERE` est court-circuitee. Ne jamais utiliser risk 3 sur un systeme en production sans autorisation ecrite.

**Trop de colonnes UNION** : si la plage par defaut (1-10) ne suffit pas, etendre a 20 ou 30 est raisonnable. Au-dela, le temps de test explose et c'est probablement un indice que l'injection n'est pas de type UNION.

## Retour terrain

En pentest, la majorite des injections tombent avec les reglages par defaut. Quand ce n'est pas le cas, c'est souvent parce que l'injection est dans un cookie (level 2 minimum) ou dans un header custom (level 3+). Avant d'augmenter les niveaux, verifie que tu cibles le bon parametre avec `-p`.

Les options `--prefix`/`--suffix` sont sous-utilisees alors qu'elles resolvent facilement les cas ou l'input est encapsule dans une fonction SQL ou une structure complexe. Prends le temps d'analyser la requete probable avant de multiplier les payloads.

Pour les cibles avec du contenu dynamique (pages qui changent a chaque chargement), `--string` est quasi indispensable. Sans cette option, SQLMap n'arrive pas a distinguer une reponse "vraie" d'une reponse "fausse" et abandonne les tests boolean-based.

## Memo express

| Besoin | Commande |
|--------|----------|
| Augmenter la couverture | `--level=3 --risk=2` |
| Couverture maximale | `--level=5 --risk=3` |
| Prefix/suffix personnalises | `--prefix="')" --suffix="-- -"` |
| Restreindre les techniques | `--technique=BEU` |
| Etendre les colonnes UNION | `--union-cols=10-20` |
| Caractere UNION personnalise | `--union-char='a'` |
| Table FROM pour UNION | `--union-from=dual` |
| Detection par chaine | `--string="Welcome"` |
| Detection par titre | `--titles` |
| Ignorer le HTML | `--text-only` |
| Detection par code HTTP | `--code=200` |

***
