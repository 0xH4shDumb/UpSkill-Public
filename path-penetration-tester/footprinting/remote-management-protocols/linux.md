# Protocoles de gestion distante Linux

La gestion à distance des systèmes Linux repose sur quelques protocoles clés. SSH est le standard, mais d'autres services comme Rsync et les anciens R-Services méritent d'être connus pour l'énumération. Chacun a ses vecteurs d'attaque propres.

## Pourquoi

En test d'intrusion, les services de gestion à distance sont souvent le chemin le plus direct vers un shell. SSH est omniprésent, Rsync peut exposer des fichiers sensibles sans authentification, et les R-Services, quand ils sont encore en place, offrent un accès presque sans restriction. Les identifier et les caractériser fait partie de la routine d'énumération.

## Comment ça marche

### SSH (Secure Shell)

SSH établit une connexion chiffrée sur le port TCP 22. OpenSSH est l'implémentation standard sur la quasi-totalité des distributions Linux.

**Méthodes d'authentification :**

- Mot de passe
- Clé publique/privée (recommandée)
- Keyboard-interactive, GSSAPI, host-based

L'authentification par clé fonctionne par défi cryptographique : le serveur envoie un challenge chiffré avec la clé publique du client, et seul le détenteur de la clé privée peut le résoudre.

**Configuration** : `/etc/ssh/sshd_config`

| Paramètre | Risque |
|---|---|
| `PasswordAuthentication yes` | Brute-force possible |
| `PermitEmptyPasswords yes` | Accès sans mot de passe |
| `PermitRootLogin yes` | Accès root direct à distance |
| `Protocol 1` | Protocole obsolète et vulnérable |

{% hint style="danger" %}
`PermitRootLogin yes` combiné à un mot de passe faible est le scénario de compromission le plus direct. En interne, beaucoup de serveurs conservent cette configuration par défaut.
{% endhint %}

### Rsync

Rsync est un outil de synchronisation de fichiers qui fonctionne sur le port TCP 873 (ou via SSH). En mode daemon, il expose des modules (partages) qui peuvent être accessibles sans authentification.

### R-Services

Suite de protocoles Unix obsolètes (rsh, rlogin, rexec, rwho, rusers) fonctionnant sur les ports TCP 512-514. Ils n'utilisent aucun chiffrement et s'appuient sur des fichiers de confiance (`/etc/hosts.equiv`, `~/.rhosts`) pour l'autorisation.

{% hint style="warning" %}
Les R-Services sont rares sur les systèmes modernes, mais on les trouve encore sur des environnements legacy ou des équipements industriels. Leur présence est toujours un finding critique.
{% endhint %}

## En pratique

### SSH

{% tabs %}
{% tab title="Audit" %}
```bash
# depuis Exegol - audit des algorithmes et de la configuration SSH
ssh-audit <IP_CIBLE>
```

ssh-audit identifie les algorithmes de chiffrement faibles, les versions obsolètes, et les problèmes de configuration. C'est l'outil de référence pour caractériser un service SSH.
{% endtab %}

{% tab title="Scan Nmap" %}
```bash
# depuis Exegol - détection et scripts NSE
sudo nmap -sV -sC -p22 <IP_CIBLE>
```
{% endtab %}

{% tab title="Connexion" %}
```bash
# depuis Exegol - connexion par mot de passe
ssh user@<IP_CIBLE>

# depuis Exegol - connexion par clé privée
ssh -i id_rsa user@<IP_CIBLE>
```
{% endtab %}
{% endtabs %}

### Rsync

```bash
# depuis Exegol - scan du service
sudo nmap -sV -p873 <IP_CIBLE>
```

```bash
# depuis Exegol - lister les modules disponibles
rsync -av --list-only rsync://<IP_CIBLE>/
```

```bash
# depuis Exegol - synchroniser un module accessible
rsync -av rsync://<IP_CIBLE>/module ./rsync-loot/
```

{% hint style="info" %}
Les modules Rsync exposés sans authentification contiennent parfois des répertoires `.ssh`, des fichiers de configuration, ou des sauvegardes. C'est un finding fréquent en environnement interne.
{% endhint %}

### R-Services

```bash
# depuis Exegol - scan des ports R-Services
sudo nmap -sV -p512,513,514 <IP_CIBLE>
```

```bash
# depuis Exegol - connexion via rlogin
rlogin <IP_CIBLE> -l root
```

## Retour terrain

SSH est le service le plus scanné et le plus ciblé, mais c'est aussi le plus souvent correctement configuré. Les vrais gains viennent des services secondaires : un Rsync sans authentification qui expose les clés SSH d'un utilisateur, un R-Service oublié qui donne un shell root sans mot de passe. Penser à inclure le port 873 et les ports 512-514 dans les scans, ils sont souvent oubliés.

## Mémo express

| Commande | Usage |
|---|---|
| `ssh-audit <IP>` | Audit de la configuration SSH |
| `ssh -i id_rsa user@<IP>` | Connexion par clé privée |
| `rsync -av --list-only rsync://<IP>/` | Lister les modules Rsync |
| `sudo nmap -sV -p22,873,512-514 <IP>` | Scan des services de gestion |
| `rlogin <IP> -l root` | Connexion R-Services |

***
