# Detection et prevention

Ce chapitre couvre les mesures defensives contre le pivoting et le tunneling. En tant que pentesters, comprendre les mecanismes de detection aide a formuler des recommandations concretes dans les rapports et a adapter nos techniques face aux environnements surveilles.

## Pourquoi

Un rapport de pentest qui se contente de lister les pivots realises sans proposer de contre-mesures est incomplet. Les clients attendent des recommandations actionnables : comment detecter ces techniques, comment les prevenir, et comment reagir en cas d'incident. Connaitre les defenses aide aussi le pentester a choisir les techniques les moins detectables selon le contexte.

## Comment ca marche

### Etablir une baseline

La detection du pivoting repose sur la capacite a distinguer le trafic anormal du trafic normal. Sans baseline, tout semble normal. Les elements essentiels a documenter :

- Inventaire des hotes et de leurs interfaces reseau
- Liste des hotes dual-homed (plusieurs NICs)
- Schema reseau a jour (outils : diagrams.net, Netbrain)
- Configuration DHCP et enregistrements DNS
- Inventaire des applications et services autorises
- Liste des utilisateurs avec des privileges eleves

### Mesures de detection

| Technique de pivoting | Indicateur de compromission | Mesure de detection |
|---|---|---|
| Tunnel SSH | Connexions SSH longue duree, volume de donnees inhabituel sur port 22 | Monitoring des sessions SSH (duree, volume) |
| Proxychains / SOCKS | Trafic TCP inhabituel depuis un hote serveur | Analyse du comportement reseau (NTA) |
| Chisel | Trafic WebSocket persistant sur HTTP | Inspection du contenu HTTP par le proxy |
| Tunneling DNS | Volume anormal de requetes DNS TXT, taille des paquets DNS inhabituelle | Monitoring DNS, detection de beaconing |
| Tunneling ICMP | Volume de paquets ICMP anormal, taille des paquets ICMP inhabituellement grande | IDS/IPS configure pour ICMP |
| Port non standard | HTTP/HTTPS sur des ports inhabituels (444 au lieu de 443) | Comparaison protocole/port attendu |

### Mesures de prevention

#### Personnes

- **MFA** (authentification multi-facteurs) sur tous les acces distants et les comptes privilegies
- **Sensibilisation** des utilisateurs aux risques du BYOD (Bring Your Own Device)
- **SOC** (Security Operations Center) avec une equipe en 24/7 pour la surveillance et la reponse aux incidents
- **Plan de reponse aux incidents** documente et regulierement teste

#### Processus

- **Gestion des acces** : provisionnement et desactivation rapides des comptes
- **Gestion des changements** : documenter qui a fait quoi et quand
- **Audits reguliers** : inventaire des hotes, verification des configurations
- **Gold images** : images systeme durcies comme baseline pour les deployements

#### Technologie

- **Segmentation reseau** : separer les reseaux de production, d'administration et d'utilisateurs
- **Firewall** : bloquer les protocoles et ports non necessaires entre les segments
- **IDS/IPS** : detecter les patterns de tunneling et les comportements anormaux
- **EDR** (Endpoint Detection and Response) : surveiller les processus et les connexions reseau sur les endpoints
- **SIEM** : correler les logs de tous les equipements pour detecter les chaines d'attaque

### Mapping MITRE ATT&CK

| TTP | Tag MITRE | Recommandation |
|---|---|---|
| Services distants externes | T1133 | Firewall perimetrique, VPN obligatoire, blocage des protocoles internes en sortie |
| Services distants (SSH, RDP) | T1021 | MFA, restriction par IP source, reseau OOB pour l'administration |
| Ports non standards | T1571 | Baseline des couples protocole/port, NTA (Network Traffic Analysis) |
| Tunneling de protocole | T1572 | Bloquer le DNS externe sauf vers les serveurs DNS autorises, inspecter le trafic HTTP |
| Utilisation de proxy | T1090 | Liste blanche de domaines/IP, proxy web avec inspection SSL |
| Living off the Land | N/A | Baseline comportementale, monitoring EDR, logs PowerShell |

{% hint style="success" %}
La mesure la plus efficace est la segmentation reseau. Un hote serveur qui ne peut communiquer qu'avec les hotes de son propre segment et le VLAN d'administration ne peut pas servir de pivot vers d'autres reseaux.
{% endhint %}

## En pratique

```bash
# Checklist de recommandations pour le rapport
# 1 - Segmentation reseau : verifier que les hotes compromis ne pouvaient pas
#     atteindre d'autres segments (si oui, recommander la segmentation)
# 2 - Monitoring : verifier que les tunnels etablis pendant le pentest
#     ont ete detectes par le SOC (si non, recommander du NTA)
# 3 - Acces distants : documenter les services SSH/RDP exposes inutilement
# 4 - DNS : verifier que les hotes internes resolvent le DNS via le serveur
#     interne et pas directement via Internet
# 5 - ICMP : documenter si les ping sont autorises entre tous les segments
```

## Retour terrain

La segmentation reseau et le monitoring sont les deux piliers de la defense contre le pivoting. En pratique, beaucoup d'entreprises ont une segmentation "plate" : tous les serveurs peuvent communiquer entre eux, et un seul pivot suffit pour atteindre le DC. Le rapport doit mettre en evidence cette faiblesse avec les preuves d'exploitation.

Les tunnels DNS et ICMP sont rarement detectes en environnement reel. La plupart des SIEM et IDS ne sont pas configures pour analyser le contenu des paquets DNS ou la taille des paquets ICMP. C'est un point important a mentionner dans le rapport.

## Memo express

| Priorite | Recommandation | Impact |
|---|---|---|
| Critique | Segmentation reseau entre les segments | Empeche le pivoting direct |
| Critique | MFA sur les acces distants | Bloque la reutilisation d'identifiants |
| Haute | Monitoring DNS (volume, TXT, taille) | Detecte le tunneling DNS |
| Haute | IDS/IPS avec regles ICMP | Detecte le tunneling ICMP |
| Haute | Baseline reseau et NTA | Detecte les comportements anormaux |
| Moyenne | Blocage SSH/RDP sortant | Empeche les tunnels SSH inverses |
| Moyenne | Inspection du trafic HTTP/HTTPS | Detecte Chisel et les C2 HTTP |

***
