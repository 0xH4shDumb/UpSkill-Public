# Methodologie et resolution de problemes

Disposer d'outils ne suffit pas. Ce qui differencie un pentester efficace d'un debutant, c'est la capacite a structurer son approche, diagnostiquer les blocages et progresser methodiquement. Cette page couvre les problemes courants, les reflexes de diagnostic et la strategie de progression.

## Pourquoi

Les debutants en pentest passent souvent des heures bloques sur un probleme qui se resout en quelques minutes avec la bonne methode de diagnostic. Savoir identifier rapidement si le probleme vient du reseau, de l'outil ou de l'approche permet de ne pas gaspiller de temps sur de fausses pistes.

## Comment ca marche

### Problemes courants et diagnostic

#### Problemes de connectivite VPN

Le VPN est le premier point de defaillance. Quand rien ne fonctionne, commencer par verifier la connexion.

| Symptome | Diagnostic | Solution |
|---|---|---|
| Pas d'interface `tun0` | VPN non connecte | Relancer `sudo openvpn client.ovpn`, verifier les logs |
| `tun0` presente mais pas de reponse | Probleme de routage | Verifier `ip route`, s'assurer que le reseau cible est route via `tun0` |
| Connexion intermittente | VPN instable | Verifier la bande passante, essayer un autre serveur VPN |
| Plusieurs interfaces `tun` | Connexions VPN multiples | Fermer toutes les connexions et n'en garder qu'une |

```bash
# - Verifier si le VPN est actif
ip -4 a show tun0

# - Verifier la table de routage
ip route | grep tun

# - Tester la connectivite vers la cible
ping -c 3 <IP_CIBLE>
```

#### La cible ne repond pas

Avant de conclure qu'une cible est hors ligne, eliminer les causes reseau.

```bash
# - Ping basique (ICMP peut etre bloque)
ping -c 3 <IP_CIBLE>

# - Scan d'un port connu (si ICMP est bloque)
nmap -Pn -p 80 <IP_CIBLE>

# - Verifier si c'est un probleme DNS
nslookup domaine.tld
```

{% hint style="info" %}
Beaucoup de cibles bloquent les requetes ICMP. L'absence de reponse au ping ne signifie pas que la machine est inaccessible. Utiliser `-Pn` avec Nmap pour scanner sans envoyer de ping prealable.
{% endhint %}

#### Un exploit ne fonctionne pas

Un exploit qui echoue n'est pas forcement inutilisable. Les causes d'echec les plus frequentes sont les suivantes.

| Cause | Verification |
|---|---|
| **Mauvaise version cible** | Verifier la version exacte du service avec `-sV` |
| **Payload incompatible** | Adapter le payload au systeme (x86 vs x64, Linux vs Windows) |
| **Firewall ou IDS** | Tester avec un payload encode ou un port different |
| **Dependances manquantes** | Lire les commentaires de l'exploit, installer les bibliotheques requises |
| **Parametres incorrects** | Verifier les chemins, IP, ports dans le code de l'exploit |

### Poser les bonnes questions

Quand on est bloque et qu'on cherche de l'aide, la qualite de la question determine la qualite de la reponse. Une question bien formulee contient quatre elements.

1. **Ou en est-on exactement** : quelle phase du pentest, quel service cible
2. **Ce qui a ete tente** : les commandes executees, les resultats obtenus
3. **Ce qui echoue** : le message d'erreur precis ou le comportement inattendu
4. **Ce qui est attendu** : le resultat escompte

{% hint style="danger" %}
Ne jamais partager de spoilers, de flags ou de solutions completes dans les canaux d'entraide. Donner des indices qui orientent vers la bonne direction sans reveler la reponse.
{% endhint %}

### Methodologie d'attaque structuree

Un pentest suit un processus iteratif. Chaque phase alimente la suivante, et il est courant de revenir a l'enumeration apres avoir obtenu un nouvel acces.

```
Enumeration → Identification de vulnerabilites → Exploitation → Post-exploitation
     ↑                                                              |
     └──────────────────────────────────────────────────────────────┘
```

#### Checklist d'approche pour une cible

1. **Scan de ports** : identifier tous les services exposes (TCP et UDP)
2. **Enumeration des services** : version, configuration, acces anonyme
3. **Enumeration web** : repertoires, fichiers, technologies, CMS
4. **Recherche d'exploits** : searchsploit, Google, CVE
5. **Exploitation** : tester les vecteurs identifies, obtenir un acces
6. **Stabilisation** : upgrade du shell, TTY interactif
7. **Enumeration locale** : utilisateurs, permissions, taches planifiees, credentials
8. **Escalade de privileges** : exploiter les misconfiguration ou vulnerabilites locales
9. **Post-exploitation** : collecter les donnees, pivoter si le perimetre le permet

{% hint style="success" %}
Si un vecteur ne fonctionne pas apres plusieurs tentatives, ne pas s'acharner. Revenir a l'enumeration et chercher un autre angle d'approche. La persistence est une qualite, l'obstination sur une piste unique est un piege.
{% endhint %}

### Documentation pendant le test

Documenter chaque etape au moment ou elle est realisee est une habitude fondamentale. Un rapport redige de memoire trois jours apres le test sera incomplet et imprecis.

Ce qu'il faut noter a chaque etape.

| Element | Exemple |
|---|---|
| **Commande executee** | `nmap -sV -sC -p- -oA full_scan 10.10.10.x` |
| **Resultat significatif** | Port 8080 ouvert, Tomcat 9.0.30 |
| **Capture d'ecran** | Preuve d'exploitation, acces obtenu |
| **Horodatage** | Date et heure de chaque action |
| **Observation** | "Le WAF bloque les requetes avec des guillemets simples" |

### Progression structuree

La montee en competences en securite offensive suit une courbe progressive. Voici un parcours type.

| Etape | Objectif | Indicateur de reussite |
|---|---|---|
| 1 | Completer des modules d'apprentissage guide | Concepts compris et appliques |
| 2 | Resoudre des machines faciles avec walkthrough | Capacite a reproduire les etapes |
| 3 | Resoudre des machines faciles sans aide | Methodologie personnelle fonctionnelle |
| 4 | Passer aux machines de difficulte moyenne | Gestion de plusieurs vecteurs d'attaque |
| 5 | Machines difficiles et labs multi-cibles | Pivoting, mouvement lateral, persistance |
| 6 | Contribuer a la communaute | Ecriture de walkthroughs, partage de connaissances |

## En pratique

### Diagnostic rapide d'un blocage

```bash
# - 1. Verifier la connectivite
ping -c 1 <IP_CIBLE> || echo "ICMP bloque"
nmap -Pn -p 80 <IP_CIBLE>

# - 2. Verifier le VPN
ip -4 a show tun0

# - 3. Verifier qu'aucun firewall local ne bloque
sudo iptables -L -n

# - 4. Relancer le scan sur un port specifique
nmap -Pn -sV -p <PORT> <IP_CIBLE>
```

### Prise de notes avec timestamps

```bash
# - Script shell pour logger les commandes avec horodatage
script -a pentest_$(date +%Y%m%d).log

# - Chaque commande sera enregistree avec sa sortie
# - Terminer avec 'exit' ou Ctrl+D
```

## Pieges et galeres

- **Tunnel vision** : rester bloque sur un seul vecteur pendant des heures. Si un exploit ne fonctionne pas apres 3-4 tentatives serieuses, changer d'approche
- **Copier-coller aveugle** : executer des commandes trouvees en ligne sans comprendre ce qu'elles font. Lire et comprendre chaque commande avant de l'executer
- **Ne pas resetter la cible** : sur les plateformes de lab, les machines peuvent etre dans un etat modifie par un autre utilisateur. Resetter la machine et recommencer si le comportement est incoherent
- **Sauter l'enumeration** : tenter d'exploiter le premier service decouvert sans enumerer completement la cible. Les vulnerabilites les plus interessantes se trouvent parfois sur un port non standard
- **Se comparer aux autres** : chacun progresse a son rythme. Ce qui compte est la comprehension des concepts, pas la vitesse de resolution

## Memo express

| Situation | Reflexe |
|---|---|
| **Cible injoignable** | Verifier VPN (`tun0`), tester avec `-Pn`, verifier la table de routage |
| **Exploit echoue** | Verifier la version cible, lire le code source, adapter les parametres |
| **Bloque depuis 30 min** | Revenir a l'enumeration, chercher un autre angle |
| **Shell instable** | Stabiliser avec `python3 pty` + `stty raw -echo` |
| **Besoin d'aide** | Formuler la question avec contexte, tentatives et erreur precise |
| **Fin de machine** | Documenter toutes les etapes, capturer les preuves |

***
