# Cassage de mots de passe

Une fois des hashs obtenus (dump de SAM, NTDS.dit, /etc/shadow, base de donnees applicative), le cassage offline permet de recuperer les mots de passe en clair. Les deux outils de reference sont Hashcat (GPU) et John the Ripper (CPU/GPU), chacun avec ses forces.

## Pourquoi

Un hash NTLM ou un hash SHA-512 de /etc/shadow n'est pas directement exploitable pour se connecter (sauf pour Pass-the-Hash). Casser le hash revele le mot de passe en clair, ce qui ouvre la porte a la reutilisation sur d'autres services, le mouvement lateral, et l'escalade de privileges.

## Comment ca marche

### Types d'attaques

| Type | Principe | Vitesse | Couverture |
|---|---|---|---|
| **Dictionnaire** | Teste chaque mot d'une wordlist | Rapide | Limitee au contenu de la liste |
| **Dictionnaire + regles** | Applique des mutations (majuscules, chiffres, symboles) | Rapide | Bien meilleure |
| **Brute force / masque** | Teste toutes les combinaisons d'un pattern | Lent pour les longs mots de passe | Complete pour le pattern |
| **Rainbow tables** | Recherche dans une table precomputee | Instantane | Inutile si salt |

### Identifier le type de hash

```bash
# - Identifier un hash avec hashid
hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
# [+] MD5 Crypt [Hashcat Mode: 500]

# - Ou via la documentation Hashcat
hashcat --help | grep -i ntlm
```

| Hash | Longueur | Exemple | Hashcat mode |
|---|---|---|---|
| MD5 | 32 hex | `e10adc3949ba59abbe56e057f20f883e` | 0 |
| SHA1 | 40 hex | `aaf4c61ddcc5e8a2...` | 100 |
| NTLM | 32 hex | `64f12cddaa88057e...` | 1000 |
| SHA-512 crypt | `$6$...` | `$6$rounds=5000$...` | 1800 |
| DCC2 | `$DCC2$...` | `$DCC2$10240#admin#...` | 2100 |

## En pratique

### Hashcat

{% tabs %}
{% tab title="Dictionnaire" %}
```bash
# - Attaque par dictionnaire simple
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt

# - Avec regles de mutation
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```
{% endtab %}
{% tab title="Masque" %}
```bash
# - Attaque par masque : Majuscule + 4 minuscules + chiffre + symbole
hashcat -a 3 -m 0 hash.txt '?u?l?l?l?l?d?s'

# Symboles de masque :
# ?l = minuscules   ?u = majuscules
# ?d = chiffres     ?s = symboles
# ?a = tout         ?1-?4 = custom
```
{% endtab %}
{% tab title="NTLM" %}
```bash
# - Casser des hashs NTLM (mode 1000)
hashcat -a 0 -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# - DCC2 (domain cached credentials, beaucoup plus lent)
hashcat -a 0 -m 2100 dcc2_hash.txt /usr/share/wordlists/rockyou.txt
```
{% endtab %}
{% endtabs %}

### John the Ripper

{% tabs %}
{% tab title="Wordlist" %}
```bash
# - Attaque par dictionnaire
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# - Avec format explicite
john --format=raw-md5 --wordlist=rockyou.txt md5_hashes.txt

# - Afficher les resultats
john --show hashes.txt
```
{% endtab %}
{% tab title="Single crack" %}
```bash
# - Mode single : genere des candidats a partir du nom d'utilisateur
# Le fichier doit contenir user:hash au format /etc/passwd
john --single passwd_file
```
{% endtab %}
{% tab title="Incremental" %}
```bash
# - Mode brute force (base sur des chaines de Markov)
john --incremental hashes.txt
```
{% endtab %}
{% endtabs %}

### Wordlists et regles personnalisees

Les wordlists generiques (rockyou.txt) ne suffisent pas toujours. En pentest, on construit souvent des wordlists ciblees a partir d'informations OSINT sur l'organisation.

```bash
# - Generer une wordlist depuis un site web avec CeWL
cewl https://www.cible.com -d 4 -m 6 --lowercase -w custom_wordlist.txt

# - Combiner des mots (attaque combinatoire)
hashcat --stdout -a 1 base.txt base.txt > combined.txt
```

#### Regles de mutation Hashcat

```bash
# - Fichier custom.rule
# : = ne rien faire
# c = premiere lettre en majuscule
# so0 = remplacer o par 0
# sa@ = remplacer a par @
# $! = ajouter ! a la fin

# - Appliquer les regles pour generer des candidats
hashcat --force base_words.txt -r custom.rule --stdout | sort -u > mutated.txt
```

| Regle | Effet sur "password" |
|---|---|
| `c` | `Password` |
| `so0` | `passw0rd` |
| `c so0` | `Passw0rd` |
| `$!` | `password!` |
| `c so0 sa@ $!` | `P@ssw0rd!` |

{% hint style="info" %}
La regle `best64.rule` incluse avec Hashcat est un excellent point de depart. Elle couvre les mutations les plus courantes (majuscules, ajout de chiffres, substitutions). Pour aller plus loin, `OneRuleToRuleThemAll.rule` est une compilation communautaire tres efficace.
{% endhint %}

## Pieges et galeres

- **Hash non identifie** : toujours utiliser `hashid` ou la documentation Hashcat avant de lancer. Un mauvais mode (`-m`) ne retournera jamais de resultat
- **DCC2 vs NTLM** : les hashs DCC2 (cached domain credentials) sont environ 800 fois plus lents a casser que les NTLM. Un mot de passe complexe en DCC2 est quasiment incassable en temps raisonnable
- **Pas de GPU** : Hashcat est concu pour le GPU. Sans GPU, John the Ripper en mode CPU est souvent plus efficace
- **Wordlist trop large** : une wordlist de 14 millions d'entrees avec 10 regles genere 140 millions de candidats. Adapter le volume au temps disponible

## Memo express

| Commande | Usage |
|---|---|
| `hashcat -a 0 -m <mode> hash wordlist` | Dictionnaire |
| `hashcat -a 3 -m <mode> hash '?u?l?l?d'` | Masque |
| `hashcat -a 0 -m <mode> hash wordlist -r rules` | Dictionnaire + regles |
| `john --wordlist=rockyou hash` | John dictionnaire |
| `john --single passwd` | John single crack |
| `john --show hash` | Afficher les resultats |
| `hashid -m <hash>` | Identifier le type de hash |
| `cewl <URL> -w wordlist.txt` | Generer une wordlist |

***
