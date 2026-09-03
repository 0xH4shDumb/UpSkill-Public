# Oracle TNS

Oracle Transparent Network Substrate (TNS) est le protocole de communication qui sert d'intermédiaire entre les applications clientes et les bases de données Oracle. On le croise fréquemment dans les environnements d'entreprise (finance, santé, grande distribution), souvent exposé sur le réseau sans que les équipes d'administration ne mesurent pleinement la surface d'attaque qu'il représente.

## Pourquoi

Oracle reste l'un des SGBD les plus déployés en environnement corporate. En test d'intrusion, tomber sur un port 1521 ouvert est une opportunité sérieuse : derrière ce listener se cache potentiellement une base de données avec des comptes par défaut, des SID devinables, et un accès aux données métier de l'entreprise.

Le protocole TNS assure la résolution de noms, la gestion des connexions, l'équilibrage de charge et le chiffrement des échanges. Il fonctionne au-dessus de TCP/IP, mais supporte aussi d'autres couches réseau. La couche Oracle Net Services qui l'enveloppe gère la totalité du cycle de vie d'une connexion client-serveur.

## Comment ça marche

### Architecture de connexion

Le listener Oracle écoute par défaut sur **TCP/1521**. Quand un client veut se connecter, il envoie un descripteur de connexion qui contient le nom du service (ou le SID) ciblé. Le listener vérifie que ce service existe, puis redirige le client vers le processus serveur approprié.

Deux fichiers de configuration contrôlent l'ensemble :

{% tabs %}
{% tab title="tnsnames.ora (client)" %}
Ce fichier, côté client, contient les alias de connexion. Chaque entrée décrit l'adresse du serveur, le port et le service cible.

```txt
ORCL =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.10.5)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

Le paramètre `SERVER` peut valoir `DEDICATED` (un processus serveur par connexion) ou `SHARED` (un pool de processus partagés, plus économe en ressources mais moins isolé).
{% endtab %}

{% tab title="listener.ora (serveur)" %}
Ce fichier configure le listener lui-même : sur quelle adresse il écoute, quels SID/services il annonce.

```txt
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = PDB1)
      (ORACLE_HOME = /opt/oracle/product/19.0.0/dbhome_1)
      (GLOBAL_DBNAME = PDB1)
    )
  )

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.10.5)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```
{% endtab %}
{% endtabs %}

Les deux fichiers se trouvent généralement dans `$ORACLE_HOME/network/admin`.

### Paramètres clés

| Paramètre | Rôle |
|---|---|
| `SERVICE_NAME` | Identifie le service de base de données ciblé |
| `SID_NAME` | Identifiant système de l'instance (souvent `XE`, `ORCL`, `PDB1`) |
| `HOST` / `PORT` | Adresse et port du listener |
| `SERVER` | Mode de connexion (`DEDICATED` ou `SHARED`) |
| `SSL_VERSION` | Version TLS utilisée pour le chiffrement |
| `TRACE_LEVEL` / `LOG_FILE` | Niveau de verbosité des logs (utile pour le debug, exploitable en reco) |

{% hint style="info" %}
La gestion à distance du listener était activée par défaut sur Oracle 8i et 9i. A partir d'Oracle 10g, elle a été désactivée, mais des installations héritées ou mal migrées peuvent encore l'exposer.
{% endhint %}

## En pratique

### Découverte du service

```bash
# Depuis Exegol - identifier le listener Oracle
sudo nmap -p1521 -sV <IP_CIBLE> --open
```

Sortie typique :

```
PORT     STATE SERVICE    VERSION
1521/tcp open  oracle-tns Oracle TNS listener 11.2.0.2.0 (unauthorized)
```

Le terme `unauthorized` signifie que le listener ne donne pas d'information complémentaire sans authentification, mais il confirme que le service est bien là.

### Enumération des SID

Le SID (System Identifier) est nécessaire pour se connecter à une instance. Si on ne le connait pas, on peut tenter de le deviner par brute-force :

```bash
# Depuis Exegol - brute-force des SID via Nmap
sudo nmap -p1521 -sV <IP_CIBLE> --script oracle-sid-brute
```

Les SID courants (`XE`, `ORCL`, `PROD`, `TEST`, `DEV`) sont souvent les premiers à tomber.

### Scan complet avec ODAT

ODAT (Oracle Database Attacking Tool) est un outil Python dédié à l'audit des bases Oracle. Il teste automatiquement les comptes par défaut, les privilèges, les possibilités d'upload de fichiers et bien plus.

```bash
# Depuis Exegol - scan complet de l'instance
odat all -s <IP_CIBLE>
```

ODAT va tester les combinaisons de credentials courantes et remonter les comptes valides avec leurs privilèges.

{% hint style="warning" %}
L'installation d'ODAT nécessite les bibliothèques Oracle Instant Client. Sur Exegol, elles sont généralement pré-installées. En dehors de cet environnement, il faut installer `instantclient-basic` et `instantclient-sqlplus`, puis configurer la variable `LD_LIBRARY_PATH`.
{% endhint %}

### Connexion avec SQLPlus

Une fois des credentials obtenus, on se connecte directement :

```bash
# Depuis Exegol - connexion standard
sqlplus <utilisateur>/<motdepasse>@<IP_CIBLE>/XE
```

Commandes utiles une fois connecté :

```sql
-- lister les tables accessibles
select table_name from all_tables;

-- vérifier les privilèges du compte
select * from user_role_privs;
```

Si le compte dispose du rôle `SYSDBA`, on peut élever la connexion :

```bash
# Depuis Exegol - connexion en tant que SYSDBA
sqlplus <utilisateur>/<motdepasse>@<IP_CIBLE>/XE as sysdba
```

Avec ce niveau de privilège, on peut extraire les hash de tous les comptes :

```sql
-- extraire les hash des mots de passe
select name, password from sys.user$;
```

### Upload de fichiers via ODAT

Si le compte dispose de droits suffisants (SYSDBA ou privilège `UTL_FILE`), ODAT permet de déposer un fichier sur le serveur :

```bash
# Depuis Exegol - upload d'un fichier sur le serveur web
echo "test" > payload.txt
odat utlfile -s <IP_CIBLE> -d XE -U <utilisateur> -P <motdepasse> --sysdba --putFile /var/www/html payload.txt ./payload.txt
```

Vérification :

```bash
curl -s http://<IP_CIBLE>/payload.txt
```

| OS | Chemin webroot habituel |
|---|---|
| Linux | `/var/www/html` |
| Windows | `C:\inetpub\wwwroot` |

## Pièges et galères

{% hint style="danger" %}
**Comptes par défaut.** Oracle est livré avec plusieurs comptes pré-configurés dont les mots de passe sont documentés publiquement : `SYS`/`CHANGE_ON_INSTALL`, `DBSNMP`/`dbsnmp`, `SCOTT`/`tiger`. En production, ces comptes ne sont pas toujours verrouillés ou modifiés.
{% endhint %}

{% hint style="warning" %}
**SID vs Service Name.** Les deux ne sont pas interchangeables. Un SID identifie une instance précise, un Service Name peut pointer vers plusieurs instances (en environnement RAC par exemple). Si la connexion échoue avec l'un, tenter l'autre.
{% endhint %}

{% hint style="warning" %}
**Instant Client manquant.** Sans les bibliothèques Oracle Instant Client correctement installées et référencées dans `LD_LIBRARY_PATH`, ni SQLPlus ni ODAT ne fonctionneront. L'erreur typique est `ORA-12162: TNS:net service name is incorrectly specified` ou un simple crash.
{% endhint %}

{% hint style="warning" %}
**Oracle 10g et versions récentes.** A partir d'Oracle 10g, les comptes par défaut sont désactivés a l'installation. Cela ne signifie pas qu'ils n'existent pas : un administrateur peut les avoir réactivés manuellement.
{% endhint %}

## Retour terrain

{% hint style="success" %}
**Toujours commencer par le SID.** Sans le SID ou le Service Name, aucune connexion n'est possible. Le brute-force de SID via Nmap est rapide et discret, c'est la première étape a systématiser.
{% endhint %}

{% hint style="success" %}
**ODAT en mode `all`.** L'option `all` d'ODAT teste l'ensemble des vecteurs en une seule passe : comptes par défaut, privilèges, upload, exécution de commandes. C'est un gain de temps considérable par rapport a une approche manuelle.
{% endhint %}

- Un accès SYSDBA donne un contrôle total sur la base : extraction de données, modification de structures, lecture de fichiers système. C'est souvent le point de bascule d'un audit.
- Sur les installations Windows, le compte Oracle tourne fréquemment avec des privilèges élevés. Un upload dans le webroot suivi d'une exécution via le navigateur peut donner un shell sur le serveur.
- Ne pas négliger les enregistrements `TRACE_LEVEL` et `LOG_FILE` dans la configuration : s'ils sont accessibles, ils peuvent révéler des requêtes SQL contenant des identifiants en clair.

## Mémo express

| Outil / Commande | Usage |
|---|---|
| `nmap -p1521 -sV --open` | Identifier le listener Oracle |
| `nmap --script oracle-sid-brute` | Brute-forcer les SID |
| `odat all -s <IP>` | Scan complet (credentials, privs, upload) |
| `sqlplus user/pass@<IP>/SID` | Connexion a la base |
| `sqlplus ... as sysdba` | Connexion avec privilèges maximaux |
| `select name, password from sys.user$` | Extraire les hash |
| `odat utlfile --putFile` | Déposer un fichier sur le serveur |

***
