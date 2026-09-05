# Securisation des applications

Ce chapitre regroupe les bonnes pratiques de durcissement pour toutes les applications couvertes dans ce module. Le pentest ne se limite pas a trouver des failles : la valeur ajoutee reside dans les recommandations concretes et priorisees que l'on fournit au client.

## Pourquoi

Les vulnerabilites exploitees dans ce module partagent des causes communes : identifiants par defaut, versions obsoletes, fonctionnalites d'administration exposees, et absence de segmentation reseau. La plupart de ces failles sont corrigeables en quelques heures avec un impact minimal sur la production. Le rapport de pentest doit presenter ces recommandations de maniere claire et actionnable.

## Comment ca marche

### Principes generaux

Cinq axes de securisation s'appliquent a toutes les applications :

1. **Authentification robuste** : changer tous les identifiants par defaut, imposer une politique de complexite, activer le MFA quand c'est possible
2. **Controle d'acces** : limiter l'acces aux interfaces d'administration par IP source (allowlist), desactiver les comptes et roles inutiles
3. **Mises a jour** : maintenir un inventaire des applications et leurs versions, appliquer les correctifs de securite dans un delai raisonnable
4. **Reduction de la surface d'attaque** : desactiver les fonctionnalites non utilisees (editeurs de templates, consoles de scripts, upload de modules, XML-RPC)
5. **Supervision** : journaliser les acces aux interfaces d'administration, alerter sur les tentatives de brute force, surveiller les modifications de fichiers

### Recommandations par application

| Application | Recommandations prioritaires |
|---|---|
| **WordPress** | Desactiver XML-RPC si non utilise. Restreindre l'acces a `/wp-admin/`. Mettre a jour les plugins et supprimer les inactifs. `DISALLOW_FILE_EDIT` dans wp-config.php |
| **Joomla** | Renommer `/administrator/`. MFA sur les comptes admin. Supprimer les extensions inutilisees. Mettre a jour regulierement |
| **Drupal** | Ne pas installer le module PHP Filter. Supprimer `CHANGELOG.txt`. Appliquer les correctifs Drupalgeddon immediatement |
| **Tomcat** | Changer les identifiants par defaut de `tomcat-users.xml`. Restreindre le Manager par IP dans `context.xml`. Desactiver AJP (port 8009) si non utilise |
| **Jenkins** | Desactiver l'acces anonyme. Restreindre la Script Console aux administrateurs. Utiliser le mode RBAC (Role-Based Access Control) |
| **Splunk** | Changer `admin:changeme` immediatement. Restreindre l'upload d'applications. Chiffrer les communications des Forwarders |
| **PRTG** | Changer `prtgadmin:prtgadmin`. Restreindre les notifications "Execute Program". Mettre a jour pour corriger CVE-2018-9276 |
| **GitLab** | Desactiver l'inscription publique si non necessaire. Forcer le 2FA. Auditer les projets "Internal" pour les secrets |
| **osTicket** | Restreindre l'acces agent par IP. Ne pas envoyer d'identifiants en clair dans les tickets |

### Integration avec l'annuaire (LDAP/AD)

{% hint style="success" %}
Quand c'est possible, integrer l'authentification des applications avec l'annuaire d'entreprise (Active Directory, LDAP). Cela permet une gestion centralisee des comptes, l'application automatique des politiques de mot de passe, et la desactivation immediate d'un compte compromis sur tous les services.
{% endhint %}

Applications supportant l'integration AD/LDAP : WordPress (plugin), Joomla (plugin), Jenkins (LDAP plugin), Splunk (natif), GitLab (natif), PRTG (natif).

### Segmentation reseau

Les interfaces d'administration ne doivent jamais etre accessibles depuis le meme reseau que les utilisateurs finaux. En pratique :

- Placer les interfaces d'administration dans un VLAN dedie
- Autoriser l'acces uniquement depuis un bastion ou un VPN d'administration
- Bloquer les ports d'administration (8080, 8443, 8009, 8089) au niveau du firewall pour les sous-reseaux non autorises

## En pratique

```bash
# Checklist post-pentest pour le rapport
# 1 - Identifier toutes les applications avec des identifiants par defaut
# 2 - Lister les versions obsoletes et les CVE associees
# 3 - Documenter les interfaces d'administration exposees
# 4 - Verifier la segmentation reseau
# 5 - Produire des recommandations priorisees (critique/haute/moyenne/basse)
```

## Retour terrain

Les recommandations les plus impactantes sont souvent les plus simples : changer les identifiants par defaut et restreindre l'acces aux interfaces d'administration. Sur la majorite des pentests internes, ces deux mesures seules auraient empeche 80% des compromissions documentees.

La difficulte n'est pas technique mais organisationnelle. Les equipes savent qu'il faut changer les mots de passe par defaut, mais les applications de monitoring et de CI/CD sont souvent deployees en urgence par l'equipe infrastructure, sans passer par le processus de securisation standard. Le rapport de pentest doit souligner ce risque de maniere concrete, avec les preuves d'exploitation, pour motiver l'action corrective.

## Memo express

| Priorite | Action | Impact |
|---|---|---|
| Critique | Changer les identifiants par defaut | Empeche les acces triviaux |
| Critique | Patcher les CVE connues | Empeche les RCE pre-auth |
| Haute | Restreindre les interfaces admin par IP | Reduit la surface d'attaque |
| Haute | Desactiver les fonctionnalites inutiles | Supprime les vecteurs d'abus |
| Moyenne | Integrer l'auth AD/LDAP | Centralise la gestion des comptes |
| Moyenne | Segmenter le reseau | Limite le mouvement lateral |
| Basse | Journaliser et alerter | Detecte les compromissions |

***
