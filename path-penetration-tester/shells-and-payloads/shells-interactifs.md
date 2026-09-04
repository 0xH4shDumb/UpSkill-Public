# Ameliorer un shell : obtenir un TTY interactif

## Pourquoi

Le shell obtenu apres exploitation est souvent restreint : pas d'autocompletion, pas de gestion des signaux (Ctrl+C tue la session au lieu de la commande), pas de prompt. On parle parfois de "jail shell". Ameliorer ce shell en un TTY interactif complet est indispensable pour travailler confortablement et eviter de perdre sa session au moindre faux mouvement.

## Comment ca marche

L'idee est d'invoquer un interpreteur de commandes complet (`/bin/bash`, `/bin/sh`) via un binaire ou un langage de script present sur la cible. Plusieurs methodes existent selon les outils disponibles.

## En pratique

### Methodes courantes de spawn de shell

{% tabs %}
{% tab title="Python" %}
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
{% endtab %}
{% tab title="/bin/sh" %}
```bash
/bin/sh -i
```
{% endtab %}
{% tab title="Perl" %}
```bash
perl -e 'exec "/bin/sh";'
```
{% endtab %}
{% tab title="Ruby" %}
```bash
ruby -e 'exec "/bin/sh"'
```
{% endtab %}
{% endtabs %}

**Autres options :**

```bash
# Lua
lua -e 'os.execute("/bin/sh")'

# AWK
awk 'BEGIN {system("/bin/sh")}'

# Find
find . -exec /bin/sh \; -quit

# Vim
vim -c ':!/bin/sh'
```

### Stabilisation complete du shell

Apres avoir obtenu un TTY basique avec Python, la stabilisation complete passe par trois etapes :

```bash
# 1. Spawn du TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'

# 2. Mise en arriere-plan (Ctrl+Z) puis configuration du terminal local
stty raw -echo; fg

# 3. Export des variables d'environnement
export TERM=xterm
```

Cette sequence donne un shell avec autocompletion, historique de commandes et gestion correcte des signaux.

### Verification des permissions

Une fois le shell ameliore, verifier immediatement les droits :

```bash
id
sudo -l
```

Un resultat comme `(ALL : ALL) NOPASSWD: ALL` indique une possibilite d'escalade directe.

## Pieges et galeres

- Python n'est pas toujours installe (conteneurs Docker minimalistes, systemes embarques). Avoir plusieurs alternatives en tete.
- La commande `stty raw -echo` modifie le terminal local. Si la session plante avant le `fg`, le terminal reste dans un etat bizarre. Taper `reset` a l'aveugle pour le restaurer.
- Sur certains systemes, `/bin/sh` pointe vers `dash` et non `bash`. Les fonctionnalites interactives y sont limitees.

## Memo express

| Outil | Commande |
|---|---|
| Python 3 | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |
| Perl | `perl -e 'exec "/bin/sh";'` |
| AWK | `awk 'BEGIN {system("/bin/sh")}'` |
| Find | `find . -exec /bin/sh \; -quit` |
| Vim | `vim -c ':!/bin/sh'` |
| Stabilisation | `stty raw -echo; fg` puis `export TERM=xterm` |

***
