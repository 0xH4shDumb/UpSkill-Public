# Infiltration Linux

## Pourquoi

Plus de 70% des serveurs web tournent sous Linux ou un derive Unix. En pentest, la compromission d'un serveur Linux est souvent le point d'entree vers le reseau interne. Les applications web hebergees sur ces serveurs constituent la surface d'attaque principale lors des engagements externes.

## Comment ca marche

L'approche suit une methodologie classique : reconnaissance, identification de la version du service, recherche de vulnerabilites connues, exploitation, obtention du shell.

### Questions a se poser

- Quelle distribution Linux est en cours d'execution ?
- Quels shells et langages de programmation sont installes ?
- Quelle application est hebergee ?
- Existe-t-il des vulnerabilites connues pour cette version ?

## En pratique

### Reconnaissance initiale

```bash
nmap -sC -sV <IP_CIBLE>
```

Les informations cles a relever : ports ouverts, versions des services, technologies web (Apache, Nginx, PHP, Python), applications identifiables sur les ports HTTP.

### Exploitation avec Metasploit

Une fois une application et sa version identifiees, rechercher les modules disponibles :

```bash
msf6 > search nom_application
```

Selectionner et configurer le module :

```bash
msf6 > use exploit/linux/http/module_exploit
msf6 > set RHOSTS <IP_CIBLE>
msf6 > set LHOST <IP_CIBLE>
msf6 > exploit
```

En cas de succes, une session Meterpreter ou un shell s'ouvre :

```bash
meterpreter > shell
```

### Exploitation manuelle

Quand Metasploit ne dispose pas du module adapte, rechercher un exploit public sur Exploit-DB ou GitHub. L'adapter au contexte (modifier les IPs, les chemins, les identifiants) et l'executer manuellement.

{% hint style="info" %}
Toujours lire et comprendre le code d'un exploit avant de l'executer. Un exploit public peut contenir du code malveillant ou des effets de bord non documentes.
{% endhint %}

## Pieges et galeres

- Les applications web n'exposent pas toujours leur version. Il faut parfois combiner plusieurs indicateurs (en-tetes HTTP, pages d'erreur, fichiers par defaut).
- Un exploit Metasploit peut ne pas fonctionner si la cible a ete patchee ou si la configuration differe de celle testee par l'auteur du module.
- L'obtention d'un shell via une application web donne souvent un compte de service (www-data, apache) avec des privileges limites. L'escalade de privileges est generalement necessaire.

## Retour terrain

En mission, l'exploitation de serveurs Linux passe le plus souvent par des failles applicatives web (upload non restreint, injection SQL, RCE via deserialization). L'exploitation directe de services systeme (SSH, FTP) est moins courante car les versions vulnerables sont rares sur des serveurs maintenus. La cle est une reconnaissance approfondie de l'application hebergee.

## Memo express

| Etape | Action |
|---|---|
| Reconnaissance | `nmap -sC -sV <IP>` |
| Identification | Version de l'application, technologies |
| Recherche | `searchsploit nom_app` ou `msf6 > search nom_app` |
| Exploitation | Module Metasploit ou exploit manuel |
| Post-exploitation | `shell`, enumeration, escalade |

***
