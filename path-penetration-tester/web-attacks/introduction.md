# Introduction aux attaques web

Une bonne partie des tests d'intrusion externes se joue sur une application web mal contrôlée plutôt que sur un service réseau exotique. Ce module regroupe trois familles de vulnérabilités qui reviennent constamment en environnement réel : le contournement de méthode HTTP, les références directes non protégées (IDOR) et l'injection XXE.

## Pourquoi ces attaques comptent

Les applications web sont devenues le point d'entrée le plus exposé de la quasi-totalité des systèmes d'information. Un portail RH, une API interne mal isolée ou un formulaire d'upload oublié suffisent souvent à obtenir un premier accès, sans avoir besoin de casser le moindre mot de passe.

Ce qui rend ces trois familles particulièrement intéressantes en pentest, c'est qu'elles ne demandent pas d'exploit sophistiqué. Elles exploitent des erreurs de conception ou de configuration : une méthode HTTP qu'on a oublié de restreindre, un identifiant d'objet qu'on expose tel quel dans une URL, un parseur XML qu'on a laissé résoudre des entités externes. Le point commun, c'est que le serveur fait exactement ce qu'on lui demande, sans vérifier que la demande est légitime.

{% hint style="danger" %}
Une application web compromise sert rarement de but final. Elle est bien souvent le pivot qui permet de rebondir vers le réseau interne : base de données accessible depuis le serveur applicatif, credentials de service récupérés dans un fichier de configuration, ou accès à des ressources internes via un SSRF déguisé en XXE.
{% endhint %}

## Les trois familles couvertes

### HTTP Verb Tampering

De nombreux serveurs et frameworks acceptent plusieurs méthodes HTTP (`GET`, `POST`, `PUT`, `DELETE`, `HEAD`...) sans que toutes soient réellement prises en compte par les contrôles de sécurité. Un développeur qui restreint l'accès à une route uniquement pour `GET` peut laisser `POST` ou `HEAD` totalement ouverts. Il suffit alors de changer le verbe de la requête pour contourner une authentification ou une règle d'autorisation qui n'a été appliquée qu'à une seule méthode.

### Insecure Direct Object References (IDOR)

C'est probablement la vulnérabilité la plus fréquente rencontrée en boîte grise ou noire. Elle apparaît quand une application expose une référence brute à une ressource (un identifiant numérique, un nom de fichier, une clé de base de données) sans vérifier que l'utilisateur qui la demande a le droit d'y accéder. En modifiant simplement un paramètre dans l'URL ou dans le corps d'une requête, on peut consulter, modifier ou supprimer les données d'un autre compte.

### Injection XXE (XML External Entity)

Quand une application traite du XML avec un parseur mal configuré ou obsolète, elle peut être amenée à résoudre des entités externes définies dans le document. Cela ouvre la porte à la lecture de fichiers locaux du serveur, au vol de credentials stockés dans des fichiers de configuration, voire dans certains cas à de l'exécution de code à distance ou du SSRF vers des services internes.

| Attaque | Cause racine | Impact typique |
|---|---|---|
| HTTP Verb Tampering | Contrôle d'accès appliqué à un seul verbe HTTP | Contournement d'authentification ou d'autorisation |
| IDOR | Absence de vérification d'autorisation sur une référence directe | Accès aux données d'autres utilisateurs |
| Injection XXE | Parseur XML autorisant la résolution d'entités externes | Lecture de fichiers, vol de credentials, RCE/SSRF |

{% hint style="info" %}
Ces trois vulnérabilités partagent une logique commune : elles n'attaquent pas la robustesse cryptographique ou la solidité d'un algorithme, mais l'absence de vérification côté serveur sur ce que le client envoie. C'est souvent le réflexe le plus simple, en test d'intrusion, qui rapporte le plus.
{% endhint %}

***
