# E5-DVSCOPC

# E5-CISSP
Zenner-Gibeaux Geoffrey
<!-- PROJECT LOGO --> <br /><h3 align="center">Projet Kubernetes</h3>

<!-- ABOUT THE PROJECT -->
## Struture du projet

 Contenu du Manifest de l'application

``` yml
# Ce fichier déploie une application Flask/Django en utilisant Infisical pour injecter les secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: rocket-configmap  # Nom de la ConfigMap (potentiellement inutile si Infisical gère toutes les variables)
data:
  DEMO_MODE: "True"  # Mode démo, variable de config non sensible
  DEBUG: "True"      # Activation du mode debug
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rocket-deployment  # Nom du déploiement
  labels:
    app: django
    env: preprod
    tier: frontend
spec:
  replicas: 1  # Nombre de pods souhaités (1 au départ)
  selector:
    matchLabels:
      app: django  # Doit correspondre aux labels du template ci-dessous
  template:
    metadata:
      labels:
        app: django
        env: preprod
        tier: frontend
      annotations:
        # ➤ Ces annotations permettent à Infisical d'injecter automatiquement les secrets depuis son backend
        infisical.com/enable: "true"  # Active l'injection de secrets via Infisical
        infisical.com/projectId: "bc385a41-05ac-42f0-b9a9-de4c79332314"  # ID unique du projet dans Infisical
        infisical.com/environment: "dev"  # Environnement Infisical ciblé
        infisical.com/secretType: "shared"  # Type de secrets à injecter (peut être "personal" ou "shared")
        infisical.com/token: "st.9d35c803-..."  # Service Token généré dans Infisical pour autoriser l'accès sécurisé
    spec:
      containers:
      - name: rocket-random  # Nom du conteneur
        image: th0m8s/rocket-app:prod  # Image Docker utilisée
        imagePullPolicy: IfNotPresent  # Ne retélécharge l’image que si elle n’est pas présente
        ports:
          - containerPort: 5005  # Port exposé par l'application
        envFrom:
        - configMapRef:
            name: rocket-configmap  # Injection des variables non sensibles depuis la ConfigMap (peut être remplacé par Infisical)
        env:
          # ➤ Variables sensibles injectées via Infisical, leurs vraies valeurs sont définies dans le dashboard Infisical
          - name: STRIPE_PUBLISHABLE_KEY
            value: ""  # Valeur injectée automatiquement par Infisical
          - name: STRIPE_SECRET_KEY
            value: ""  # Idem
        resources:
          requests:
            memory: "64Mi"  # Réservation mémoire minimale
            cpu: "250m"     # Réservation CPU minimale
          limits:
            memory: "120Mi" # Limite mémoire
            cpu: "300m"     # Limite CPU
---
apiVersion: v1
kind: Service
metadata:
  name: rocket-service  # Nom du service exposant l'application
spec:
  type: LoadBalancer  # Type de service pour être exposé à l'extérieur (ex: cloud provider)
  selector:
    app: django  # Doit matcher avec les labels du pod
  ports:
  - port: 5005       # Port du service
    targetPort: 5005 # Port du conteneur
    name: django-np  # Nom du port (utilisé dans l'Ingress)
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-rocketapp  # Nom de l'ingress
spec:
  defaultBackend:
    service:
      name: rocket-service
      port:
        number: 7777  # Redirige tout le trafic vers le service sur ce port (custom, à adapter si besoin)
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rocket-hpa  # HPA (autoscaling horizontal)
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rocket-deployment  # Cible le déploiement défini plus haut
  minReplicas: 1  # Nombre minimum de pods
  maxReplicas: 5  # Nombre maximum de pods
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Si le CPU dépasse 50%, Kubernetes ajoute plus de pods


```

 Contenu du Manifest de secret

``` yml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: infisical-secret-crd
  labels:
    label-to-be-passed-to-managed-secret: sample-value
  annotations:
    example.com/annotation-to-be-passed-to-managed-secret: "sample-value"
spec:
  hostAPI: http://192.168.49.2:32397/api
  # hostAPI: http://infisical-backend.infisical.svc.cluster.local:4000/api
  resyncInterval: 10
  authentication:
    universalAuth:
      secretsScope:
        projectSlug: sec-ops-ur-yt
        envSlug: dev
        secretsPath: "/"
      credentialsRef:
        secretName: infisicalsecret-universalauth-crd
        secretNamespace: default
  managedSecretReference:
    secretName: managed-secret
    secretNamespace: default
    creationPolicy: "Orphan"
    template:
      includeAllSecrets: true
      data:
        NEW_KEY_NAME: "{<!-- -->{ .KEY.SecretPath }} {<!-- -->{ .KEY.Value }}"
        KEY_WITH_BINARY_VALUE: "{<!-- -->{ .KEY.SecretPath }} {<!-- -->{ .KEY.Value }}"
```
 Contenu du Manifest infisical-secret-crd

``` yml
# ➤ Ce fichier définit une ressource personnalisée (CRD) de type InfisicalSecret.
# Elle permet à Infisical de créer automatiquement un Secret Kubernetes basé sur les variables stockées dans Infisical.

apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret  # CRD (Custom Resource Definition) fournie par Infisical
metadata:
  name: infisical-secret-crd  # Nom de cette ressource Infisical
  labels:
    label-to-be-passed-to-managed-secret: sample-value  # Étiquette propagée au Secret généré
  annotations:
    example.com/annotation-to-be-passed-to-managed-secret: "sample-value"  # Annotation propagée au Secret généré
spec:
  hostAPI: http://192.168.49.2:32397/api
  # hostAPI: http://infisical-backend.infisical.svc.cluster.local:4000/api
  # ➤ Adresse de l'API Infisical. Peut être une IP locale (comme ici), ou le service Infisical interne au cluster.
  
  resyncInterval: 10
  # ➤ Re-synchronise les secrets toutes les 10 secondes avec Infisical.
  
  authentication:
    universalAuth:
      secretsScope:
        projectSlug: sec-ops-ur-yt  # Slug du projet dans Infisical
        envSlug: dev               # Environnement ciblé (dev, staging, prod…)
        secretsPath: "/"          # Chemin racine pour récupérer tous les secrets
      credentialsRef:
        secretName: infisicalsecret-universalauth-crd  # Nom du Secret contenant les credentials Infisical
        secretNamespace: default                       # Namespace dans lequel ce Secret est stocké
  
  managedSecretReference:
    secretName: managed-secret          # Nom du Secret Kubernetes à créer/mettre à jour
    secretNamespace: default            # Namespace dans lequel le Secret sera créé
    creationPolicy: "Orphan"            # Si ce CRD est supprimé, le Secret reste ("Orphan" = ne pas supprimer)
    template:
      includeAllSecrets: true           # Tous les secrets présents dans le scope seront injectés
      data:
        # ➤ Exemple de transformation personnalisée (non obligatoire)
        NEW_KEY_NAME: "{{ .KEY.SecretPath }} {{ .KEY.Value }}"
        KEY_WITH_BINARY_VALUE: "{{ .KEY.SecretPath }} {{ .KEY.Value }}"

```
Contenu du Manifest infisicalsecret-universalauth-crd

``` yml
# ➤ Ce fichier contient les identifiants chiffrés permettant l’authentification à Infisical
# utilisés dans le fichier précédent via `credentialsRef`.

apiVersion: v1
kind: Secret
metadata:
  name: infisicalsecret-universalauth-crd  # Ce nom doit correspondre à celui référencé dans le CRD précédent
type: Opaque  # Type standard pour un secret arbitraire
data:
  clientId: MThlNmI3NGYt...  # Identifiant client (encodé en base64)
  clientSecret: OTg5MDJhMTg5...  # Secret client (encodé en base64)

```

Contenu du Manifest simple-values-exemple

``` yml
# ➤ Ce fichier est un exemple d’un Secret statique Kubernetes,
# utile pour comparer avec l’approche dynamique d’Infisical.

apiVersion: v1
kind: Secret
metadata:
  name: infisical-secrets  # Nom du Secret dans Kubernetes
type: Opaque  # Indique que le contenu est arbitraire (non lié à des certificats ou credentials spécifiques)
stringData:
  AUTH_SECRET: ABCgDHjNqmyCMRHLuv6XdeZRjgrS0WlYmsPuf4qnzow=  # Secret utilisé pour l’authentification (en clair ici)
  ENCRYPTION_KEY: 3f668a19385b7636c9d00ea2bc3dd14e            # Clé de chiffrement
  SITE_URL: http://192.168.49.2                               # URL du site (non sensible)
  HTTPS_ENABLED: "false"                                      # Configuration (non sensible)
  TELEMETRY_ENABLED: "false"                                  # Désactivation de la télémétrie

```
Contenu du Manifest values

``` yml
# ➤ Fichier de valeurs utilisé pour un déploiement Helm (probablement pour déployer Infisical lui-même)
# Permet de configurer l'image et certains paramètres comme l'ingress nginx.

infisical:
  image:
    repository: infisical/infisical  # Image Docker officielle
    tag: "v0.97.0-postgres"          # Version avec support PostgreSQL
    pullPolicy: IfNotPresent         # Politique de récupération de l’image Docker
ingress:
  nginx:
    enabled: true  # Active le support Ingress avec nginx (important si Infisical est exposé via une URL)


```
<!-- GETTING STARTED -->

### Installer Infisical en self host dans k8s via helm


1 - Déploiement du secret avant le déploiement d’infisical
k create -f simple-values-example.yaml

Déploiement d’infisical
helm upgrade --install infisical infisical-helm-charts/infisical-standalone --values values.yaml

La webinterface d’infisical est généralement accessible via le port 80. Pour s’en assurer, afficher l’@ip et le port
k get service infisical-ingress-nginx-controller

Prendre le node port et faire un port forwarding

Pour ensuite y accéder, réaliser un port forwarding via vscodeium ou un k
OU on peut aussi y accéder directement via l’@ip de la vm, faire un `kubectl port-foward` et choisir le service de
l’app en question et bien mettre les ports de l’app des deux côtés

kubectl port-forward --address 0.0.0.0 services/infisical-ingress-nginx-controller port:port

2 - Installation de l'opérateur infisical dans k8s

🔗 https://infisical.com/docs/integrations/platforms/kubernetes/overview

On va maintenant installer l’opérateur infisical dans k8s qui nous permettra de récupérer et injecter des secrets :

# https://infisical.com/docs/integrations/platforms/kubernetes/overview
# Déploiement
helm repo add infisical-helm-charts 'https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/' && 
helm repo update && helm install --generate-name infisical-helm-charts/secrets-operator

3 - Connexion de l’opérateur k8s infisical avec le serveur infisical

🔗 https://infisical.com/docs/documentation/platform/identities/overview

   Création d’un CRD (définition de ressources personnalisées) qui permettra à Infisical de se connecter à k8s et d’utiliser les secrets d’Infisical en tant que secrets k8s :

   Génération d’un ID machine (client id) et d’un client secret

   Test via curl ou cli

   Création d’un manifest secret avec les infos (client id, client secret)
    
   Création d’un manifest CRD qui piochera les secrets dans celui précédemment créé


<p align="right"><a href="#readme-top">back to top</a></p>
