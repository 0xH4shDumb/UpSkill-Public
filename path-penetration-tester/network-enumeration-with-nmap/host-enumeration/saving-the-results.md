# Sauvegarde des résultats

## Pourquoi

Un scan sans trace écrite, c'est du travail perdu. En pentest, tu enchaînes les phases — reconnaissance, énumération, exploitation — et tu as besoin de revenir sur tes résultats à chaque étape. Sauvegarder systématiquement tes scans te permet de :

- **Comparer** les résultats entre plusieurs passes (avant/après modification de paramètres)
- **Documenter** proprement tes découvertes pour le rapport final
- **Reprendre** là où tu t'es arrêté sans tout rescanner
- **Partager** tes résultats avec un coéquipier ou un client dans un format lisible

Sans fichier de sortie, tu te retrouves à rescanner des cibles déjà traitées, et ça fait perdre un temps précieux — surtout sur des plages réseau larges.

## Comment ça marche

Nmap propose trois formats de sortie, chacun adapté à un usage différent :

| Format | Option | Extension | Usage principal |
|--------|--------|-----------|-----------------|
| Normal | `-oN` | `.nmap` | Lecture humaine, copier-coller rapide |
| Grepable | `-oG` | `.gnmap` | Filtrage avec `grep`, `awk`, `cut` — idéal pour le scripting |
| XML | `-oX` | `.xml` | Parsing automatisé, génération de rapports HTML, import dans d'autres outils |

L'option `-oA` combine les trois d'un coup : tu donnes un préfixe, et Nmap génère les trois fichiers correspondants. C'est le réflexe à adopter par défaut.

## En pratique

### Sauvegarder dans les trois formats simultanément

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -p- -oA scan-initial
```

Trois fichiers sont créés dans le répertoire courant :

```bash
ls scan-initial.*
scan-initial.nmap   scan-initial.gnmap   scan-initial.xml
```

### Format normal (`.nmap`)

C'est la sortie telle que tu la verrais dans le terminal, enregistrée dans un fichier texte :

```bash
# Depuis Exegol
cat scan-initial.nmap
```

```
# Nmap 7.93 scan initiated as: nmap -p- -oA scan-initial <IP_CIBLE>
Nmap scan report for <IP_CIBLE>
Host is up (0.009s latency).
Not shown: 65532 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
80/tcp open  http

# Nmap done -- 1 IP address (1 host up) scanned in 10.22 seconds
```

### Format grepable (`.gnmap`)

Tout tient sur une ligne par hôte — parfait pour extraire les ports ouverts avec un one-liner :

```bash
# Depuis Exegol
cat scan-initial.gnmap
```

```
# Nmap 7.93 scan initiated as: nmap -p- -oA scan-initial <IP_CIBLE>
Host: <IP_CIBLE> ()	Status: Up
Host: <IP_CIBLE> ()	Ports: 22/open/tcp//ssh///, 25/open/tcp//smtp///, 80/open/tcp//http///	Ignored State: closed (65532)
# Nmap done -- 1 IP address (1 host up) scanned in 10.22 seconds
```

Tu peux ensuite filtrer rapidement :

```bash
# Depuis Exegol — extraire uniquement les ports ouverts
grep "open" scan-initial.gnmap | cut -d"/" -f1
```

### Format XML (`.xml`)

Le XML est verbeux mais structuré. Son intérêt principal : la conversion en rapport HTML.

```bash
# Depuis Exegol
cat scan-initial.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE nmaprun>
<nmaprun scanner="nmap" args="nmap -p- -oA scan-initial" version="7.93">
  <host>
    <address addr="<IP_CIBLE>" addrtype="ipv4"/>
    <ports>
      <port protocol="tcp" portid="22">
        <state state="open" reason="syn-ack"/>
        <service name="ssh"/>
      </port>
      <!-- ... -->
    </ports>
  </host>
</nmaprun>
```

### Générer un rapport HTML depuis le XML

L'outil `xsltproc` applique la feuille de style XSL intégrée à Nmap pour produire un rapport visuel :

```bash
# Depuis Exegol
xsltproc scan-initial.xml -o scan-initial.html
```

Le fichier HTML obtenu est lisible dans n'importe quel navigateur — propre à envoyer à un client ou à intégrer dans un rapport.

## Pièges & galères

- **Oublier `-oA` et perdre ses résultats** : prends le réflexe de toujours ajouter `-oA` suivi d'un nom explicite. C'est trois lettres qui t'évitent de rescanner pendant 20 minutes.
- **Noms de fichiers vagues** : `scan1`, `test`, `output`… Au bout de trois cibles, tu ne sais plus quoi est quoi. Adopte une convention : `<cible>-<type>-<date>`, par exemple `webapp-full-20250903`.
- **Écraser un fichier existant** : Nmap écrase sans prévenir si le préfixe existe déjà. Vérifie avant de lancer, ou utilise un suffixe incrémental.
- **xsltproc absent** : sur certaines images minimales, `xsltproc` n'est pas installé. Sur Exegol, il est disponible nativement.

## Retour terrain

En mission, je sauvegarde systématiquement avec `-oA` dès le premier scan. Le format grepable est mon allié pour scripter l'enchaînement : extraire les ports ouverts, les passer à un scan de version ciblé, puis alimenter un script d'énumération automatique. Le XML, lui, sert surtout en fin de mission pour générer les annexes du rapport. L'habitude de nommer proprement ses fichiers de sortie paraît anodine, mais sur un périmètre de 500 machines, c'est ce qui fait la différence entre un pentest organisé et un chaos de fichiers illisibles.

## Mémo express

| Besoin | Commande |
|--------|----------|
| Sauvegarder dans tous les formats | `nmap <IP_CIBLE> -oA <prefixe>` |
| Sortie texte uniquement | `nmap <IP_CIBLE> -oN resultat.nmap` |
| Sortie grepable uniquement | `nmap <IP_CIBLE> -oG resultat.gnmap` |
| Sortie XML uniquement | `nmap <IP_CIBLE> -oX resultat.xml` |
| Convertir XML → HTML | `xsltproc resultat.xml -o resultat.html` |
| Extraire les ports ouverts (grepable) | `grep "open" resultat.gnmap` |

***
