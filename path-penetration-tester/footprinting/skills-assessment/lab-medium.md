# Lab - Serveur interne multi-services (NFS, SMB, RDP)

## Scenario

On te confie l'audit d'un second serveur. Celui-ci est accessible a l'ensemble des utilisateurs du reseau interne, ce qui en fait une cible de choix pour un attaquant : les serveurs partages en interne sont souvent moins durcis que ceux exposes sur Internet. Le client a accepte de l'inclure dans le perimetre apres qu'on lui a explique le risque.

Aucun identifiant n'est fourni cette fois. L'objectif est de partir de zero, d'enumerer les services, de trouver des informations exploitables dans les partages accessibles, puis de pivoter jusqu'a obtenir un acces authentifie.

## Approche

Sur un serveur Windows interne, on s'attend a voir des services de partage (SMB, NFS) et potentiellement du RDP. La strategie suit une logique d'escalade informationnelle :

1. **Scan des services** pour identifier les protocoles exposes
2. **Enumeration des partages NFS et SMB** en acces anonyme ou avec des sessions nulles
3. **Analyse du contenu** des fichiers trouves (tickets de support, logs, configurations)
4. **Reutilisation des identifiants** decouverts sur d'autres services (SMB, RDP, bases de donnees)

{% hint style="info" %}
Les tickets de support technique sont une source frequente de fuites d'identifiants. Les utilisateurs y collent regulierement des extraits de configuration contenant des mots de passe en clair.
{% endhint %}

## Commandes

### Scan initial

```bash
sudo nmap -sV -sC <IP_CIBLE> -oN footprint_medium_tcp
```

On repere les services cles : RPC/NFS (ports 111, 2049), SMB (ports 139, 445), RDP (port 3389).

### Enumeration NFS

Si NFS est present, on verifie les partages exportes :

```bash
showmount -e <IP_CIBLE>
```

Montage du partage :

```bash
mkdir target-NFS
sudo mount -t nfs <IP_CIBLE>:/<PARTAGE> ./target-NFS -o nolock
```

Une fois monte, on identifie les fichiers non vides :

```bash
# - lister les fichiers avec leur taille pour reperer ceux qui contiennent des donnees
sudo ls -al target-NFS/
```

{% hint style="success" %}
Dans un repertoire contenant des dizaines de fichiers, commencer par trier par taille (`ls -alS`) permet d'aller droit aux fichiers qui ont du contenu. Les fichiers de 0 octet sont des leurres ou des fichiers vides.
{% endhint %}

### Analyse du contenu

```bash
# - lire les fichiers non vides
sudo cat target-NFS/<FICHIER_INTERESSANT>
```

On cherche des informations d'identification : noms d'utilisateurs, mots de passe, configurations de services (SMTP, bases de donnees, etc.).

### Pivot vers SMB

Avec les identifiants trouves, on enumere les partages SMB :

```bash
smbclient -U <UTILISATEUR> -L //<IP_CIBLE>
```

Connexion a un partage specifique :

```bash
smbclient -U <UTILISATEUR> //<IP_CIBLE>/<PARTAGE>
```

```bash
smb: \> ls
smb: \> get <FICHIER>
```

### Pivot vers les bases de donnees

Si les fichiers SMB contiennent d'autres identifiants (par exemple pour un compte `sa` SQL Server), on peut se connecter via RDP et acceder a la base de donnees :

```bash
xfreerdp /v:<IP_CIBLE> /u:<UTILISATEUR> /p:'<MOT_DE_PASSE>' /cert:ignore
```

{% tabs %}
{% tab title="MSSQL via sqlcmd" %}
```bash
sqlcmd -S localhost -U sa -P '<MOT_DE_PASSE>'
```

```sql
SELECT name FROM sys.databases;
GO

USE <DATABASE>;
GO

SELECT * FROM <TABLE>;
GO
```
{% endtab %}

{% tab title="MSSQL via impacket" %}
```bash
# - depuis Exegol, sans RDP
impacket-mssqlclient sa:'<MOT_DE_PASSE>'@<IP_CIBLE>
```

```sql
SELECT name FROM master.dbo.sysdatabases;
GO

USE <DATABASE>;
GO

SELECT * FROM <TABLE>;
GO
```
{% endtab %}
{% endtabs %}

## Ce qu'on en retient

- NFS en environnement Windows interne est un signal d'alerte. Les partages NFS sont souvent moins controles que SMB et permettent un acces anonyme par defaut.
- Les tickets de support sont une mine d'or pour l'attaquant. Les utilisateurs partagent regulierement des configurations avec des identifiants en clair dans des conversations de support.
- La reutilisation de mots de passe entre services est courante. Un mot de passe SMTP peut fonctionner sur SMB, et un compte `sa` SQL Server est souvent accessible avec des identifiants trouves dans des fichiers de configuration.
- Le cheminement typique en interne suit une logique de rebond : partage anonyme, puis identifiants, puis acces authentifie, puis escalade vers des services plus sensibles (bases de donnees, RDP).

***
