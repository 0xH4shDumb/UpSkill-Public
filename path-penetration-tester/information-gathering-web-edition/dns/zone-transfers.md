# Transferts de zone DNS

## Pourquoi

Le transfert de zone DNS (AXFR) est un mécanisme conçu pour synchroniser les enregistrements entre un serveur DNS primaire et ses secondaires. En temps normal, c'est une opération d'administration parfaitement légitime. Mais quand un serveur est mal configuré et accepte les transferts de zone de n'importe qui, il livre d'un coup l'intégralité de sa base DNS. Pour un pentester, c'est le jackpot : tous les sous-domaines, toutes les IPs, tous les services, en une seule requête.

## Comment ça marche

### Le mécanisme légitime

Un transfert de zone permet à un serveur DNS secondaire de récupérer une copie complète des enregistrements depuis le serveur primaire. Ce processus garantit :

- La **résilience** du service DNS (si le primaire tombe, les secondaires prennent le relais).
- La **cohérence** des données entre tous les serveurs autoritaires.
- Une résolution DNS fiable et rapide pour les utilisateurs finaux.

### Le risque

Historiquement, les serveurs DNS acceptaient les requêtes AXFR sans restriction. Aujourd'hui, c'est considéré comme une **faille de configuration majeure**, mais on en trouve encore régulièrement en audit. Un transfert de zone non sécurisé expose :

- La **liste complète des sous-domaines** : services internes, interfaces d'administration, environnements de test.
- Les **adresses IP associées** à chaque entrée.
- Les **enregistrements MX** qui révèlent les serveurs mail internes.
- Les **serveurs NS**, qui fournissent des indices sur l'infrastructure réseau.

{% hint style="danger" %}
Un transfert de zone réussi revient à obtenir la carte complète de l'infrastructure DNS de la cible. C'est l'une des premières choses à tester lors d'une reconnaissance DNS.
{% endhint %}

## En pratique

### Tenter un transfert de zone avec dig

La commande est simple. Il faut connaître le serveur DNS autoritaire (récupérable via `dig <domaine> NS`) et le domaine cible :

```bash
# - Identifier les serveurs NS
dig <DOMAINE_CIBLE> NS +short

# - Tenter le transfert de zone
dig axfr @<SERVEUR_NS> <DOMAINE_CIBLE>
```

#### Exemple sur un serveur vulnérable

```bash
dig axfr @<IP_CIBLE> cible.htb
```

Si le transfert réussit, la sortie ressemble à quelque chose comme :

```
cible.htb.             604800  IN  SOA   cible.htb. root.cible.htb. 2 604800 86400 2419200 604800
cible.htb.             604800  IN  NS    ns.cible.htb.
admin.cible.htb.       604800  IN  A     10.10.34.2
ftp.admin.cible.htb.   604800  IN  A     10.10.34.2
internal.cible.htb.    604800  IN  A     127.0.0.1
dev.ir.cible.htb.      604800  IN  A     10.10.45.6
cible.htb.             604800  IN  SOA   cible.htb. root.cible.htb. 2 604800 86400 2419200 604800
```

On obtient d'un coup la totalité des enregistrements de la zone : sous-domaines, IPs internes, services cachés. Le nombre d'enregistrements est indiqué en fin de sortie (`XFR size: N records`).

### Que faire avec les résultats

- **Cartographier les sous-domaines** : identifier les services internes (`admin.`, `internal.`, `dev.`).
- **Repérer les réseaux internes** : les adresses en `10.x.x.x`, `172.16.x.x` ou `192.168.x.x` révèlent la segmentation réseau.
- **Identifier les services** : les noms comme `ftp.`, `mail.`, `vpn.` indiquent les services disponibles.
- **Prioriser les cibles** : les sous-domaines `dev`, `staging`, `test` sont souvent moins protégés que la production.

## Pièges et galères

- **Refus quasi systématique** : sur les infrastructures modernes bien configurées, les transferts de zone sont restreints aux IPs des serveurs secondaires légitimes. Un refus se manifeste par un message `Transfer failed` ou une absence de réponse.
- **Faux négatif** : tester tous les serveurs NS du domaine. Un serveur peut refuser le transfert alors qu'un autre l'accepte.
- **Périmètre d'engagement** : le transfert de zone est une technique active qui peut être logguée. S'assurer qu'elle est dans le scope de la mission.

## Retour terrain

Les transferts de zone non sécurisés deviennent de plus en plus rares, mais ils existent encore, surtout dans les environnements internes ou chez les organisations qui n'ont pas audité leur configuration DNS récemment. C'est un test rapide qui prend quelques secondes et qui, quand il fonctionne, fait gagner des heures de reconnaissance.

## Mémo express

| Besoin | Commande |
|---|---|
| Lister les serveurs NS | `dig <domaine> NS +short` |
| Tenter un transfert de zone | `dig axfr @<serveur_ns> <domaine>` |
| Compter les enregistrements | Observer `XFR size: N records` en fin de sortie |

***
