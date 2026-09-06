# Durcissement Linux

Apres avoir passe un module entier a exploiter des failles de configuration et des vulnerabilites, il est essentiel de comprendre comment s'en proteger. Un systeme correctement durci elimine la majorite des vecteurs d'escalade presentes dans les pages precedentes.

## Pourquoi

Le durcissement n'est pas un exercice theorique. Chaque recommandation de cette page repond a un vecteur d'attaque concret couvert dans ce module. Un pentester qui comprend les mesures defensives peut evaluer leur mise en place lors d'un audit, identifier les lacunes, et formuler des recommandations actionnables dans son rapport.

## Comment ca marche

### Mises a jour et correctifs

Les exploits les plus simples ciblent des versions connues de logiciels vulnerables. Un noyau non patche est vulnerable a Dirty Pipe, un sudo non mis a jour est vulnerable a Baron Samedit, un polkit ancien est vulnerable a PwnKit. Les mises a jour regulieres eliminent ces vecteurs.

| Distribution | Outil de mise a jour automatique |
|---|---|
| Ubuntu / Debian | `unattended-upgrades` (installe par defaut depuis Ubuntu 18.04) |
| Red Hat / CentOS | `yum-cron` ou `dnf-automatic` |

{% hint style="info" %}
La mise a jour automatique des paquets de securite est la mesure defensive la plus impactante. Elle elimine a elle seule la majorite des kernel exploits et des CVE recentes sans intervention manuelle.
{% endhint %}

### Gestion des configurations

La plupart des escalades exploitent des erreurs de configuration, pas des vulnerabilites zero-day. Les mesures suivantes couvrent les vecteurs les plus courants.

| Mesure | Vecteur couvert |
|---|---|
| Auditer les binaires SUID/SGID | Permissions speciales |
| Specifier les chemins absolus dans les cron jobs et sudoers | Manipulation de PATH |
| Ne pas stocker de credentials en clair dans des fichiers accessibles | Credential hunting |
| Nettoyer les repertoires home et l'historique bash | Fuite d'informations |
| Verifier que les bibliotheques custom ne sont pas modifiables | Shared object hijacking |
| Supprimer les paquets et services inutiles | Surface d'attaque |
| Activer SELinux ou AppArmor | Controle d'acces supplementaire |

### Gestion des utilisateurs

La surface d'attaque est directement proportionnelle au nombre de comptes et de privileges distribues.

| Mesure | Detail |
|---|---|
| **Limiter les comptes** | Minimiser le nombre de comptes locaux et administrateurs |
| **Surveiller les connexions** | Logger et monitorer les tentatives de connexion (valides et invalides) |
| **Politique de mots de passe** | Privilegier les passphrases longues plutot que les rotations frequentes |
| **Historique des mots de passe** | Utiliser `/etc/security/opasswd` avec PAM pour empecher la reutilisation |
| **Groupes** | Ne pas placer les utilisateurs dans des groupes qui leur donnent des privileges excessifs (`docker`, `lxd`, `disk`) |
| **Sudo** | Appliquer le principe du moindre privilege dans les regles sudoers |

### Gestion de la configuration automatisee

Pour les environnements avec de nombreux serveurs, l'automatisation est indispensable.

| Outil | Usage |
|---|---|
| **Puppet** | Gestion de configuration declarative |
| **SaltStack** | Automatisation et orchestration |
| **Ansible** | Configuration management agentless |
| **Zabbix** | Monitoring et verification de checksums |
| **Nagios** | Monitoring avec actions de remediation |

Ces outils permettent de deployer des verifications automatiques (checksums de binaires critiques, permissions SUID, ports en ecoute) et d'alerter en cas de deviation.

### Audit de securite

Les audits periodiques completent les verifications automatisees. Plusieurs referentiels fournissent des bases de comparaison.

| Referentiel | Organisme | Usage |
|---|---|---|
| **DISA STIGs** | DoD | Guides techniques de securite par OS |
| **CIS Benchmarks** | CIS | Configurations recommandees par plateforme |
| **ISO 27001** | ISO | Cadre de gestion de la securite de l'information |
| **PCI DSS** | PCI SSC | Requis pour le traitement de donnees de paiement |

## En pratique

### Audit automatise avec Lynis

Lynis est un outil open source qui audite la configuration d'un systeme Unix et fournit des recommandations de durcissement.

```bash
# - Cloner et executer Lynis
git clone https://github.com/CISOfy/lynis.git
cd lynis
./lynis audit system

# Le rapport couvre :
# - Authentification et mots de passe
# - Permissions de fichiers
# - Pare-feu et reseau
# - Services en cours
# - Mises a jour disponibles
# - Score global de durcissement
```

### Checklist de durcissement rapide

```bash
# - Auditer les binaires SUID (comparer avec une baseline)
find / -perm -4000 -type f 2>/dev/null | sort > /tmp/suid_current.txt

# - Verifier les scripts de cron pour les chemins relatifs
grep -r "[^/]bin/" /etc/crontab /etc/cron.d/ 2>/dev/null

# - Chercher les fichiers world-writable hors /tmp et /proc
find / -xdev -perm -0002 -type f -not -path "/proc/*" -not -path "/tmp/*" 2>/dev/null

# - Verifier les permissions des repertoires dans PATH
echo $PATH | tr ':' '\n' | xargs ls -ld 2>/dev/null

# - Chercher des credentials en clair
grep -rni "password\|passwd\|secret" /etc/ /opt/ /var/www/ 2>/dev/null | head -20

# - Verifier l'etat de SELinux ou AppArmor
getenforce 2>/dev/null
aa-status 2>/dev/null

# - Verifier les mises a jour disponibles
apt list --upgradable 2>/dev/null
```

### Recommandations post-audit typiques

| Constat | Recommandation |
|---|---|
| Binaires SUID non standards | Retirer le bit SUID ou appliquer des capabilities granulaires |
| Cron jobs avec chemins relatifs | Specifier les chemins absolus (`/usr/bin/tar` au lieu de `tar`) |
| Scripts world-writable executes par root | Restreindre les permissions (`chmod 700`) |
| Utilisateurs dans le groupe docker | Retirer du groupe ou utiliser Docker rootless |
| Sudo NOPASSWD trop large | Restreindre aux commandes specifiques necessaires |
| Noyau non patche | Planifier les mises a jour automatiques |
| SELinux en mode permissive | Passer en mode enforcing |

{% hint style="success" %}
Les audits automatises (Lynis, CIS-CAT) sont des complements, pas des remplacements pour un pentest. Un scan Lynis ne detectera pas une chaine d'exploitation complexe. En revanche, il identifie rapidement les ecarts par rapport aux bonnes pratiques.
{% endhint %}

## Pieges et galeres

- **Durcissement excessif** : des permissions trop restrictives peuvent casser des applications. Tester chaque changement dans un environnement de pre-production
- **SELinux en enforcing** : le passage de permissive a enforcing peut bloquer des services legitimes. Verifier les logs d'audit (`/var/log/audit/audit.log`) et creer des politiques adaptees
- **SUID necessaires** : certains binaires SUID sont essentiels (`passwd`, `su`, `mount`). Ne pas les supprimer aveuglement
- **Mises a jour et compatibilite** : sur les systemes legacy, une mise a jour du noyau peut casser des applications dependantes de versions specifiques
- **Faux sentiment de securite** : un bon score Lynis ne garantit pas la securite. Les audits doivent etre completes par des tests d'intrusion reguliers

## Memo express

| Outil / Action | Usage |
|---|---|
| `unattended-upgrades` | Mises a jour automatiques (Debian/Ubuntu) |
| `yum-cron` | Mises a jour automatiques (RHEL/CentOS) |
| `./lynis audit system` | Audit de securite automatise |
| `find / -perm -4000` | Auditer les SUID |
| `getenforce` | Statut SELinux |
| `aa-status` | Statut AppArmor |
| Puppet / Ansible / Salt | Gestion de configuration automatisee |
| CIS Benchmarks / DISA STIGs | Referentiels de durcissement |

***
