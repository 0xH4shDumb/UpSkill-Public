# Introduction aux shells et payloads

## Pourquoi

Obtenir un shell sur une cible, c'est le moment ou l'on passe de la theorie a la pratique. C'est l'acces interactif au systeme de fichiers, aux processus, aux configurations. Sans shell, on ne fait que de la reconnaissance. Avec un shell, on enumere, on escalade, on pivote.

Un shell peut etre obtenu de plusieurs facons : exploitation d'une vulnerabilite, upload de fichier malveillant, injection de commande. Le payload est le code qui rend cet acces possible. Comprendre comment les shells fonctionnent et comment les payloads les delivrent est une competence fondamentale en pentest.

## Comment ca marche

### Qu'est-ce qu'un shell

Un shell est un programme qui fournit une interface en ligne de commande pour interagir avec un systeme d'exploitation. Sur Linux, les plus courants sont Bash, Zsh et sh. Sur Windows, cmd.exe et PowerShell.

En pentest, on parle de "shell" au sens large : un acces distant interactif a la ligne de commande d'un systeme compromis.

### Anatomie d'un shell

Un environnement CLI se compose de trois elements :

| Element | Role |
|---|---|
| Emulateur de terminal | Programme qui affiche le terminal (MATE Terminal, Windows Terminal, kitty, Alacritty) |
| Interpreteur de commandes | Programme qui execute les commandes (Bash, PowerShell, cmd) |
| Systeme d'exploitation | Couche sous-jacente qui traite les appels systeme |

L'emulateur de terminal n'est pas lie a un interpreteur specifique. On peut lancer PowerShell dans MATE Terminal, ou Bash dans Windows Terminal (via WSL).

**Identifier le shell actif :**

{% tabs %}
{% tab title="Linux" %}
```bash
ps
# ou
echo $SHELL
```
{% endtab %}
{% tab title="Windows" %}
```powershell
$PSVersionTable
# ou
echo %COMSPEC%
```
{% endtab %}
{% endtabs %}

### Qu'est-ce qu'un payload

En securite offensive, un payload est le code execute apres exploitation d'une vulnerabilite. C'est la charge utile qui transforme une faille en acces. Un payload peut etre aussi simple qu'un one-liner Netcat ou aussi complexe qu'un implant Meterpreter multi-stage.

Le payload n'est pas l'exploit. L'exploit exploite la vulnerabilite, le payload est ce qui s'execute une fois la porte ouverte.

## En pratique

Ce module couvre l'ensemble du cycle :

- **Types de shells** : bind shell, reverse shell, web shell
- **Payloads** : concepts, staged vs stageless, generation avec MSFvenom
- **Amelioration de shell** : passage d'un shell basique a un TTY interactif
- **Infiltration Linux** : exploitation d'applications vulnerables sur serveurs Linux
- **Infiltration Windows** : fingerprinting, exploitation (EternalBlue), formats de payloads
- **Web shells** : PHP, ASPX (Laudanum, Antak), deploiement et utilisation
- **Automatisation** : Metasploit pour la livraison de payloads
- **Detection et prevention** : comprendre le volet defensif

## Retour terrain

L'erreur classique en debut de parcours est de se focaliser sur l'obtention du shell sans comprendre ce qu'on execute. Copier-coller un reverse shell depuis une cheat sheet sans savoir ce que fait chaque partie de la commande, c'est dangereux en mission (risque de crash du service, traces non maitrisees). Decomposer et comprendre ses payloads fait la difference entre un operateur fiable et quelqu'un qui tire dans le tas.

## Memo express

| Concept | Definition |
|---|---|
| Shell | Interface CLI interactive avec un systeme |
| Payload | Code execute apres exploitation d'une vulnerabilite |
| Bind shell | La cible ecoute, l'attaquant se connecte |
| Reverse shell | L'attaquant ecoute, la cible se connecte |
| Web shell | Shell accessible via navigateur (fichier PHP/ASPX/JSP) |
| Staged | Payload envoye en plusieurs etapes |
| Stageless | Payload autonome, envoye en une fois |

***
