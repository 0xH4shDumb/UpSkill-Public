# Lab — Contourner un pare-feu pour identifier un service

## Scénario

Tu fais face à une machine dont certains ports hauts sont protégés par un pare-feu. Un scan classique te montre un port dans l'état `tcpwrapped` — autrement dit, Nmap arrive à établir la connexion TCP mais le service coupe immédiatement sans envoyer de bannière. Impossible d'en tirer la moindre information avec un scan standard.

L'objectif : identifier la version exacte du service qui tourne derrière ce port filtré.

## Approche

La démarche se décompose en trois temps :

1. **Scan complet des ports** — Lancer un `-p-` avec détection de version (`-sV`) pour repérer tous les ports ouverts et isoler celui qui apparaît comme `tcpwrapped`.
2. **Bypass par source-port 53** — Beaucoup de pare-feux laissent passer le trafic en provenance du port 53 (DNS). En forçant ton port source à 53, tu peux traverser le filtrage là où un port source aléatoire serait bloqué.
3. **Banner grabbing manuel** — Quand Nmap ne suffit pas, `ncat` permet de se connecter directement au port et de récupérer la bannière à la main, en spécifiant le port source.

## Commandes

### Étape 1 — Scan complet avec détection de version

```bash
# Depuis Exegol
sudo nmap <IP_CIBLE> -p- -sS -sV -Pn -n --disable-arp-ping --source-port 53
```

Tu devrais obtenir une liste de ports avec la plupart identifiés (SSH, HTTP…), et un port haut marqué `tcpwrapped`. C'est celui qu'on cible.

### Étape 2 — Connexion manuelle avec ncat

Le port source 53 est la clé. On l'utilise directement avec `ncat` pour récupérer la bannière :

```bash
# Depuis Exegol — banner grabbing via port source DNS
ncat -nv --source-port 53 <IP_CIBLE> 50000
```

Si le bypass fonctionne, la connexion s'établit et le service t'envoie sa bannière — là où Nmap affichait `tcpwrapped`, tu obtiens maintenant l'identité réelle du service.

### Vérification alternative avec Nmap + script banner

Tu peux aussi combiner le script `banner` avec le port source 53 dans Nmap :

```bash
sudo nmap <IP_CIBLE> -p 50000 -sS -sV -Pn -n --disable-arp-ping --source-port 53 --script=banner
```

## Ce qu'on en retient

- **`tcpwrapped` ≠ impasse** — Ça signifie juste que le service ferme la connexion avant d'envoyer quoi que ce soit à Nmap. Le service est bien là, il refuse simplement de parler à n'importe qui.
- **Le port source 53 est un passe-partout classique** — Les administrateurs oublient souvent de filtrer le trafic provenant du port DNS. C'est une des premières techniques d'évasion à tester.
- **`ncat` complète Nmap** — Quand les scripts et la détection automatique échouent, une connexion manuelle avec `ncat` donne souvent le résultat que Nmap n'a pas pu obtenir. Pense toujours à vérifier manuellement les ports suspects.
- **Combine les outils** — Un scan Nmap pour la cartographie globale, puis `ncat` (ou `nc`) pour les cas particuliers. C'est cette approche en couches qui fait la différence sur le terrain.

***
