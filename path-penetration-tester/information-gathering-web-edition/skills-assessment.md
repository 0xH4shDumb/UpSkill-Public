# Lab - Reconnaissance web

## Scénario

Ce lab met en pratique l'ensemble des techniques de reconnaissance web abordées dans ce module. L'objectif est d'analyser un domaine cible en combinant plusieurs approches : WHOIS, DNS, fingerprinting, crawling, brute-force de VHosts, et analyse des résultats collectés.

## Approche

La méthodologie suit une progression logique :

1. **WHOIS** : identifier le registrar et les informations d'enregistrement du domaine principal.
2. **Fingerprinting** : déterminer le serveur web et les technologies en place.
3. **Énumération de VHosts** : découvrir des sous-domaines non référencés dans le DNS public.
4. **Analyse des fichiers exposés** : consulter `robots.txt` pour identifier des répertoires cachés.
5. **Crawling** : explorer les sous-domaines découverts pour collecter emails, commentaires et autres données.
6. **Énumération récursive** : brute-forcer les sous-domaines des VHosts découverts pour trouver des niveaux supplémentaires.

## Commandes

### Étape 1 : WHOIS

```bash
# - Identifier le registrar et l'ID IANA
whois <DOMAINE_CIBLE>
```

Les champs à relever : `Registrar`, `Registrar IANA ID`, `Creation Date`, `Name Servers`.

### Étape 2 : Fingerprinting HTTP

```bash
# - Identifier le serveur web
curl -I http://<DOMAINE_CIBLE>:<PORT>
```

Le header `Server` dans la réponse révèle le logiciel et sa version.

### Étape 3 : Énumération de VHosts

```bash
# - Brute-force des VHosts
gobuster vhost -u http://<DOMAINE_CIBLE>:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain -t 50
```

Chaque VHost découvert doit être ajouté au fichier `/etc/hosts` :

```bash
echo '<IP_CIBLE>    <VHOST_DECOUVERT>' >> /etc/hosts
```

### Étape 4 : Analyse de robots.txt

```bash
# - Consulter le robots.txt des VHosts découverts
curl -s http://<VHOST_DECOUVERT>:<PORT>/robots.txt
```

Les chemins en `Disallow` peuvent pointer vers des répertoires sensibles (panels admin, fichiers de configuration).

### Étape 5 : Crawling

```bash
# - Crawler les sous-domaines découverts
python3 ReconSpider.py http://<VHOST_DECOUVERT>:<PORT>

# - Analyser les résultats
cat results.json | jq .
```

Examiner attentivement les champs `emails` et `comments` du JSON. Les commentaires HTML contiennent parfois des credentials, des clés API ou des informations de migration.

### Étape 6 : Énumération récursive

Si le crawling initial ne donne rien, chercher des sous-domaines supplémentaires basés sur les VHosts déjà trouvés :

```bash
# - Brute-force récursif sur un VHost découvert
gobuster vhost -u http://<VHOST_DECOUVERT>:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain -t 50
```

Ajouter les nouveaux VHosts au `/etc/hosts` et relancer le crawling dessus.

## Ce qu'on en retient

- La reconnaissance web est un processus itératif. Chaque découverte peut ouvrir de nouvelles pistes à explorer.
- Le brute-force de VHosts et le crawling se complètent : le premier découvre les hôtes, le second en extrait le contenu.
- Les commentaires HTML et le fichier `robots.txt` sont des sources d'information souvent sous-estimées qui méritent une attention systématique.
- L'ajout des VHosts au `/etc/hosts` est une étape technique indispensable pour pouvoir interagir avec les hôtes découverts.

***
