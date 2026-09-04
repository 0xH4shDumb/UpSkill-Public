# Filtrage des resultats

## Pourquoi

Un scan brut avec ffuf peut retourner des centaines, voire des milliers de resultats. Sans filtrage, le signal est noye dans le bruit. ffuf offre des mecanismes de correspondance (matchers) et de filtrage (filters) pour isoler les reponses pertinentes.

## Comment ca marche

ffuf propose deux approches complementaires :

- **Matchers** (`-m*`) : garder uniquement les reponses qui correspondent a un critere
- **Filters** (`-f*`) : exclure les reponses qui correspondent a un critere

Par defaut, ffuf utilise le matcher `-mc 200,204,301,302,307,401,403` (codes HTTP courants). On peut ajouter des filtres supplementaires pour affiner.

### Options de filtrage

| Option | Type | Filtre sur |
|---|---|---|
| `-mc` / `-fc` | Match / Filter | Code HTTP |
| `-ms` / `-fs` | Match / Filter | Taille de la reponse (bytes) |
| `-mw` / `-fw` | Match / Filter | Nombre de mots |
| `-ml` / `-fl` | Match / Filter | Nombre de lignes |
| `-mr` / `-fr` | Match / Filter | Expression reguliere |

## En pratique

### Filtrer par taille de reponse

Le cas le plus courant : toutes les reponses invalides retournent la meme taille (ex: 900 bytes pour une page par defaut). On filtre cette taille :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/ -H 'Host: FUZZ.domaine.htb' -fs 900
```

### Filtrer par code HTTP

Exclure les reponses 403 (acces interdit) si elles ne sont pas pertinentes :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/FUZZ -fc 403
```

Ou ne garder que les 200 :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/FUZZ -mc 200
```

### Matcher "all" + filtre

Pour voir tout sauf un pattern specifique, utiliser `-mc all` avec un filtre :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/FUZZ -mc all -fs 42
```

### Filtrer par nombre de mots ou de lignes

Utile quand la taille varie legerement mais le nombre de mots reste constant :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/FUZZ -fw 20
```

### Filtrer par regex

Pour exclure les reponses contenant un message d'erreur specifique :

```bash
ffuf -w list:FUZZ -u http://<IP_CIBLE>/FUZZ -fr "Not Found"
```

## Pieges et galeres

- Ne jamais lancer un scan sans verifier d'abord la taille de reponse par defaut. Un premier passage rapide (quelques dizaines d'entrees) suffit pour identifier le pattern a filtrer.
- Les plages sont supportees dans les filtres : `-fs 100-200` exclut toutes les tailles entre 100 et 200 bytes.
- `-mc all` desactive le filtrage par defaut sur les codes HTTP. A combiner avec un filtre explicite, sinon les resultats sont inutilisables.
- Attention aux redirections (301/302) : elles peuvent fausser la taille de reponse. Ajouter `-follow` si necessaire, mais ca ralentit le scan.

## Retour terrain

Le filtrage est ce qui rend le fuzzing exploitable. La regle d'or est : lancer un scan rapide, observer la taille et le code des reponses par defaut, configurer le filtre, puis relancer. En deux passes, on obtient des resultats propres et exploitables. En engagement, documenter les filtres utilises dans les notes de scan pour pouvoir les reproduire.

## Memo express

| Objectif | Option |
|---|---|
| Exclure une taille | `-fs <taille>` |
| Exclure un code | `-fc <code>` |
| Garder uniquement un code | `-mc <code>` |
| Exclure un nombre de mots | `-fw <nombre>` |
| Matcher tout + filtre | `-mc all -fs <taille>` |
| Filtre regex | `-fr "<pattern>"` |

***
