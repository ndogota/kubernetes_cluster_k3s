Objectif du projet
------------------

Dans le cadre de la mise en place d’une nouvelle solution d’hébergement, nous utiliserons Kubernetes pour l’orchestration des services, avec K3S en tant que Rancher.

Prérequis
---------

Nous avons provisionné trois machines virtuelle sur le cloud DigitalOcean.

Chacune des ces machine virtuelle tourne sur Ubuntu 23.10 x64 et possède ces caractéristiques : 2 vCPUs - 2GB Ram / 60GB Disk

Ces trois machine virtuelles sont accessible au travers de ces ip's publique :

```text-plain
ubuntu-s-2vcpu-2gb-fra1-01	167.172.179.89
ubuntu-s-2vcpu-2gb-fra1-02	167.172.188.255
ubuntu-s-2vcpu-2gb-fra1-03	167.172.189.215
```

Ces trois machine virtuelle sont en mesure de communiquer entre elles par le biais d'un VPC (Virtual Private Cloud).

Installation de K3S
-------------------

Connectez-vous à chaque serveurs via SSH.

Mettez à jour les packages avec les commandes suivantes (à exécuter sur toutes les VMs):

```text-plain
sudo apt-get update
sudo apt-get upgrade -y
```

### Installation de K3S sur le Master

Connectez-vous à k3s-master via SSH.

Exécutez la commande d'installation de K3S:

```text-plain
curl -sfL https://get.k3s.io | sh -
```

Vérifiez que K3S est bien installé et que le cluster est initialisé :

```text-plain
sudo kubectl get nodes
```

Sur le master, récupérez le token nécessaire pour joindre les nœuds workers au cluster :

```text-plain
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Joindre les Workers au Cluster

Connectez-vous à chaque worker via SSH.

Utilisez la commande suivante pour joindre chaque worker au cluster, en remplaçant YOUR\_MASTER\_IP par l'IP du master et YOUR\_TOKEN par le token récupéré précédemment :

```text-plain
curl -sfL https://get.k3s.io | K3S_URL=https://YOUR_MASTER_IP:6443 K3S_TOKEN=YOUR_TOKEN sh -
```

Retournez sur votre master et vérifiez que les workers sont bien connectés :

```text-plain
sudo kubectl get nodes
```

Sur le master, vérifiez si Traefik est installé :

```text-plain
sudo kubectl get pods --namespace kube-system | grep traefik
```

Déploiement de Wordpress avec MariaDB (ou MySQL)
------------------------------------------------

Nous allons déployer Wordpress avec MariaDB étape par étape.

Tout d'abord, vous devez créer un namespace dédié pour isoler les ressources de votre application Wordpress. Créez un fichier appelé wordpress-namespace.yaml avec le contenu suivant :

```text-plain
apiVersion: v1
kind: Namespace
metadata:
 name: wordpress
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f wordpress-namespace.yaml
```

Vous allez stocker les identifiants de la base de données dans un objet Secret pour les garder sécurisés.

Créez un fichier wordpress-secrets.yaml :

```text-plain
apiVersion: v1
kind: Secret
metadata:
 name: wordpress-secrets
 namespace: wordpress
type: Opaque
data:
 mysql-root-password: <encoded-root-password>
 mysql-user: <encoded-user>
 mysql-password: <encoded-password>
```

Remplacez <encoded-root-password>, <encoded-user>, et <encoded-password> par des valeurs encodées en base64.

Vous pouvez générer ces valeurs encodées avec la commande :

```text-plain
echo -n 'your-password' | base64
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f wordpress-secrets.yaml
```

Créez un fichier mariadb-deployment.yaml pour le déploiement de MariaDB :

```text-plain
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mariadb
 namespace: wordpress
spec:
 selector:
   matchLabels:
     app: mariadb
 strategy:
   type: Recreate
 template:
   metadata:
     labels:
       app: mariadb
   spec:
     containers:
     - name: mariadb
       image: mariadb:latest
       env:
       - name: MYSQL_ROOT_PASSWORD
         valueFrom:
           secretKeyRef:
             name: wordpress-secrets
             key: mysql-root-password
       - name: MYSQL_DATABASE
         value: wordpress
       - name: MYSQL_USER
         valueFrom:
           secretKeyRef:
             name: wordpress-secrets
             key: mysql-user
       - name: MYSQL_PASSWORD
         valueFrom:
           secretKeyRef:
             name: wordpress-secrets
             key: mysql-password
       ports:
       - containerPort: 3306
       volumeMounts:
       - name: mariadb-storage
         mountPath: /var/lib/mysql
     volumes:
     - name: mariadb-storage
       persistentVolumeClaim:
         claimName: mariadb-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: mariadb-pvc
 namespace: wordpress
spec:
 accessModes:
 - ReadWriteOnce
 resources:
   requests:
     storage: 10Gi
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f mariadb-deployment.yaml
```

Créez un fichier wordpress-deployment.yaml pour le déploiement de Wordpress :

```text-plain
apiVersion: apps/v1
kind: Deployment
metadata:
 name: wordpress
 namespace: wordpress
spec:
 selector:
   matchLabels:
     app: wordpress
 strategy:
   type: Recreate
 template:
   metadata:
     labels:
       app: wordpress
   spec:
     containers:
     - name: wordpress
       image: wordpress:latest
       env:
       - name: WORDPRESS_DB_HOST
         value: mariadb
       - name: WORDPRESS_DB_USER
         valueFrom:
           secretKeyRef:
             name: wordpress-secrets
             key: mysql-user
       - name: WORDPRESS_DB_PASSWORD
         valueFrom:
           secretKeyRef:
             name: wordpress-secrets
             key: mysql-password
       - name: WORDPRESS_DB_NAME
         value: wordpress
       ports:
       - containerPort: 80
       volumeMounts:
       - name: wordpress-storage
         mountPath: /var/www/html
     volumes:
     - name: wordpress-storage
       persistentVolumeClaim:
         claimName: wordpress-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: wordpress-pvc
 namespace: wordpress
spec:
 accessModes:
 - ReadWriteOnce
 resources:
   requests:
     storage: 10Gi
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f wordpress-deployment.yaml
```

Créez un fichier wordpress-service.yaml pour exposer Wordpress :

```text-plain
apiVersion: v1
kind: Service
metadata:
 name: wordpress
 namespace: wordpress
spec:
 type: ClusterIP
 ports:
   - port: 80
 selector:
   app: wordpress
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f wordpress-service.yaml
```

Créez un fichier mariadb-service.yaml pour exposer Wordpress :

```text-plain
apiVersion: v1
kind: Service
metadata:
 name: mariadb
 namespace: wordpress
spec:
 ports:
 - port: 3306
   targetPort: 3306
 selector:
   app: mariadb
```

Appliquez ce manifeste avec la commande :

```text-plain
sudo kubectl apply -f mariadb-service.yaml
```

Sécurisation des accès Web
--------------------------

Finalement, vous allez créer une règle Ingress pour rendre votre site Wordpress accessible via HTTP. Créez un fichier wordpress-ingress.yaml avec le contenu suivant :

```text-plain
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: wordpress-ingress
 namespace: wordpress
 annotations:
   kubernetes.io/ingress.class: "traefik"
spec:
 rules:
 - host: "wordpress.your-domain.com"
   http:
     paths:
     - path: /
       pathType: Prefix
       backend:
         service:
           name: wordpress
           port:
             number: 80
```

Appliquez ce fichier avec la commande :

```text-plain
sudo kubectl apply -f wordpress-ingress.yaml
```

Nous avons récupérer le fichier ConfigMap, pour le TLS par LetsEncrypt :

```text-plain
apiVersion: v1
kind: ConfigMap
metadata:
 name: traefik-config
 namespace: kube-system
 labels:
   app: traefik-ingress
data:
 traefik.toml: |
   # Global settings
   [global]
   checkNewVersion = false
   sendAnonymousUsage = false
   # Servers transport settings
   [serversTransport]
   insecureSkipVerify = true
   # Entrypoints settings
   [entryPoints]
   [entryPoints.http]
   address = ":80"
   [entryPoints.https]
   address = ":443"
   [entryPoints.traefik]
   address = ":8080"
   [entryPoints.monitoring]
   address = ":9090"
   # Logs settings
   [log]
   level = "ERROR"
   format = "json"
   [accessLog]
   format = "json"
   # API dashboard settings
   [api]
   dashboard = true
   debug = false
   insecure = true
   # Monitoring settings
   [ping]
   entryPoint = "monitoring"
   # Providers settings
   [providers]
   [providers.file]
   directory = "/custom/"
   watch = true
   [providers.kubernetesCRD]
   # Metrics settings
   [metrics]
   [metrics.prometheus]
   addEntryPointsLabels = true
   addServicesLabels = true
   entryPoint = "traefik"
   # Certificates settings
   [certificatesResolvers]
   [certificatesResolvers.letsencrypt]
   [certificatesResolvers.letsencrypt.acme]
   email = "your-email@example.com"  # Change this to your email address
   storage = "/certs/letsencrypt.json"
   caServer = "https://acme-v02.api.letsencrypt.org/directory"
   [certificatesResolvers.letsencrypt.acme.httpChallenge]
   entryPoint = "http"
   # Use the staging environment for testing; comment this out in production
   [certificatesResolvers.letsencrypt-staging]
   [certificatesResolvers.letsencrypt-staging.acme]
   email = "your-email@example.com"  # Change this to your email address
   storage = "/certs/letsencrypt-staging.json"
   caServer = "https://acme-staging-v02.api.letsencrypt.org/directory"
   [certificatesResolvers.letsencrypt-staging.acme.httpChallenge]
   entryPoint = "http"
---
apiVersion: v1
kind: ConfigMap
metadata:
 name: traefik-custom
 namespace: kube-system
 labels:
   app: traefik-ingress
data:
 tls.toml: |
   # TLS settings
   [tls]
   [tls.options]
   [tls.options.default]
   minVersion = "VersionTLS12"
   [tls.options.mintls13]
   minVersion = "VersionTLS13"
 middlewares.toml: |
   # Middlewares settings
   [http.middlewares]
   # Redirect to HTTPS
   [http.middlewares.redirect-https.redirectScheme]
   scheme = "https"
   permanent = true
   # Remove default headers (security)
   [http.middlewares.default-headers.headers]
   [http.middlewares.default-headers.headers.customResponseHeaders]
   X-Powered-By = ""
   Server = ""
   # Add default headers (security)
   [http.middlewares.security-headers.headers]
   AccessControlAllowOrigin = "null"
   AccessControlMaxAge = 60
   BrowserXssFilter = true
   ContentTypeNoSniff = true
   ForceSTSHeader = true
   FrameDeny = true
   SSLRedirect = true
   STSIncludeSubdomains = true
   STSPreload = true
   STSSeconds = 315360000
   CustomFrameOptionsValue = "SAMEORIGIN"
   ReferrerPolicy = "same-origin"
   FeaturePolicy = "vibrate 'self'"
```

Après avoir mis à jour le ConfigMap, vous pouvez l'appliquer à votre cluster avec la commande suivante :

```text-plain
sudo kubectl apply -f traefik-configmap.yaml
```

Nous pouvons modifier notre Ingress pour utiliser le TLS :

```text-plain
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
 name: wordpress-https
 namespace: wordpress
spec:
 entryPoints:
   - websecure  # Cela fait référence à l'entrée sécurisée HTTPS dans la configuration de Traefik
 routes:
 - match: Host(`your-wordpress-domain.com`)  # Remplacez avec votre domaine réel
   kind: Rule
   services:
   - name: wordpress
     port: 80
 tls:
   certResolver: letsencrypt  # Assurez-vous que ceci correspond au nom de votre certResolver dans Traefik
```

Amélioration de la gestion des secrets
--------------------------------------

Avant d'installer Vault, vous devez d'abord installer Helm, qui est un gestionnaire de packages pour Kubernetes. Voici comment installer Helm :

Téléchargez le script d'installation de Helm :

```text-plain
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Vérifiez que Helm est correctement installé :

helm version  
 

Une fois Helm installé, vous devez ajouter le dépôt de HashiCorp à votre configuration Helm :

```text-plain
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Mettez à jour la liste des charts pour récupérer les dernières versions disponibles :

```text-plain
helm repo update
```

Installez Vault dans votre cluster Kubernetes en utilisant Helm. Par défaut, Vault est installé en mode "dev" qui n'est pas recommandé pour la production. Pour une utilisation en production, vous devrez fournir une configuration personnalisée :

```text-plain
helm install vault hashicorp/vault --create-namespace --namespace vault
```

Maintenant vous aller installer helm et kubectl sur votre machine en local, pour cela vous aller definir la version de Vault que vous voulez utiliser et recuperer le binaire :

```text-plain
VAULT_VERSION="1.9.0" # Remplacez ceci par la dernière version de Vault
wget https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip
```

Si vous n'avez pas unzip installé sur votre système, installez-le avec :

```text-plain
sudo apt-get install unzip
```

Décompressez le binaire de Vault :

```text-plain
unzip vault_${VAULT_VERSION}_linux_amd64.zip
```

Déplacez le binaire de Vault dans un répertoire qui fait partie de votre PATH. /usr/local/bin est un choix commun :

```text-plain
sudo mv vault /usr/local/bin/
```

Assurez-vous que le binaire de Vault est exécutable :

```text-plain
sudo chmod +x /usr/local/bin/vault
```

Vérifiez que Vault est maintenant installé correctement :

```text-plain
vault version
```

Dans le cadre d'un developpement simplifier, vous pouvez installer kubectl sur votre machine en local et copier le fichier de configuration de k3s de votre master vers votre host, une fois cela effectuer vous pourrez export le fichier dans une variable d'environnement : 

```text-plain
export KUBECONFIG=/chemin/vers/votre/config
```

Une fois le client CLI de Vault installé, vous pouvez vous connecter à votre instance Vault dans Kubernetes en définissant l'adresse du serveur Vault et en utilisant le token d'authentification. Vous pouvez obtenir l'adresse du serveur Vault en exécutant :

```text-plain
kubectl port-forward --namespace vault svc/vault 8200:8200
```

Et dans un autre terminal, vous définissez la variable d'environnement VAULT\_ADDR :

```text-plain
export VAULT_ADDR='http://127.0.0.1:8200'
```

Si vous n'avez pas encore initialisé Vault, faites-le avec la commande suivante :

```text-plain
vault operator init
```

Vault démarre dans un état "sealed", ce qui signifie qu'il est verrouillé et ne peut pas accéder à ses données. Vous devez fournir un certain nombre de clés de déverrouillage (par défaut, 3 sur 5) pour "unseal" Vault et le rendre opérationnel.

Utilisez la commande suivante pour chaque clé de déverrouillage :

```text-plain
vault operator unseal
```

Et maintenant, vous pouvez essayer de vous connecter à nouveau en utilisant :

```text-plain
# Connectez-vous à Vault (il faudra d'abord récupérer le root token)
vault login
```

Lorsque vous êtes invité à saisir un token, entrez le token d'accès initial que vous avez obtenu lors de l'initialisation.

Pour configurer la rotation des secrets, vous devrez interagir avec l'interface CLI de Vault ou l'interface utilisateur Web pour configurer les politiques et les authentifications nécessaires. Voici un exemple général de commandes CLI Vault pour configurer la rotation des secrets :

```text-plain
# Créez une politique pour permettre la rotation des secrets
vault policy write my-policy - <<EOF
# Configuration de la politique ici
EOF

# Configurez un secret engine qui supporte la rotation des secrets
vault secrets enable -path=my-secrets kv-v2

# Configurez une authentification qui permet l'auto-rotation
# Dépend du type d'authentification que vous utilisez
```

Observabilité
-------------

Prometheus est un système de surveillance et d'alerte tandis que Grafana est une plateforme pour la visualisation des données de surveillance.

Utilisez Helm pour installer Prometheus dans votre cluster. Prometheus collectera les métriques de votre cluster et de vos applications :

```text-plain
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus
```

Utilisez Helm pour installer Grafana, qui vous permettra de visualiser les métriques collectées par Prometheus :

```text-plain
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana
```

Après l'installation, obtenez le mot de passe administrateur pour Grafana avec la commande suivante :

```text-plain
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Configurez le port forwarding pour accéder à l'interface web de Grafana :

```text-plain
kubectl port-forward --namespace default svc/grafana 3000:80
```

Prometheus écoute généralement sur le port 9090 dans le pod. Pour faire suivre ce port à votre machine locale :

```text-plain
kubectl port-forward --namespace default svc/prometheus-server 9090:80
```

Utilisez Helm pour installer Loki dans votre cluster :

```text-plain
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack
```

Promtail sera installé avec le stack Loki si vous utilisez le chart loki-stack. Promtail sera configuré pour collecter les logs et les envoyer à Loki.

Déploiement d’un service de documentation
-----------------------------------------

Pour simplifier le déploiement nous allons installer le service de documentation directement sur notre noeud master, cela nous évitera de build notre image docker sur une architecture différente de notre noeud master.

Exécutez les commandes suivantes pour installer Docker :

```text-plain
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Une fois l'installation terminée, vérifiez que Docker fonctionne :

```text-plain
sudo docker version
```

Connectez-vous à votre serveur k3s-master.

Créer un Nouveau Site Hugo :

```text-plain
hugo new site my-documentation
cd my-documentation
```

Choisissez un thème à partir de Hugo Themes et ajoutez-le à votre site. Par exemple :

```text-plain
git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
echo 'theme = "ananke"' >> config.toml
```

Créez du contenu pour votre site. Par exemple :

```text-plain
hugo new posts/my-first-post.md
```

Lancez le serveur Hugo en local pour tester :

```text-plain
hugo server -D
```

Créez un Dockerfile dans le répertoire racine de votre projet Hugo qui ressemble à ceci :

```text-plain
# Etape de build
FROM klakegg/hugo:0.81.0-ext-alpine AS builder
COPY . /src
RUN hugo --minify --source /src
# Etape de production
FROM nginx:alpine
COPY --from=builder /src/public /usr/share/nginx/html
```

Construisez l'image Docker avec la commande suivante :

```text-plain
docker build -t mon-site-hugo .
```

Avant de pouvoir pousser l'image sur Docker Hub, assurez-vous d'être connecté à Docker Hub dans votre terminal :

```text-plain
docker login
```

Entrez votre nom d'utilisateur et votre mot de passe Docker Hub lorsque vous y êtes invité.

Une fois connecté, poussez l'image avec :

```text-plain
docker push monusername/hugo-site:latest
```

Voici comment utiliser port-forward avec votre service Hugo en dehors de votre cluster :

```text-plain
kubectl port-forward svc/hugo-service 8080:80
```
