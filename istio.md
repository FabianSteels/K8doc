ISTIO
==

Installation
--
L'installation de fait en 3 phases:
* requirement (helm + k8)
* installation de la chart initiale d'istio (CustomResourceDefinition)
* installation des composants d'Istio dans un namespace spécifique

Principes d'Istio
--
Chaque `Pod` est associé avec un `sidecar`(Envoy proxy).

Les `sidecar` communiquent avec un composant d'Istio (`Mixer`). Ce composant permet de centraliser la télémetrie (metrics, log, traces) et les policy.

Le composant `Pilot` permet de centraliser toutes les données de configuration des side-car.

Le composant `Citadel` permet de gérer les certificats (authentification par certificat possible) 

Les `sidescar` envoient des métriques télémétrie: requête recues, mémoire, cpu, erreur, ...) au mixer qui aggrége ces métriques à l'aide de `prometheus` (port 9090). `Grafana`est un outil de visualisation qui se branche sur `prometheus`(port 3000) avec des dashboards déjà pré-établi

### Custom Resource Definition et Istio

Istio est un ensemble de Custom Resource Definition proposant une extention à l'API de Kubernetes.

Voici une description des quelques CRD les plus utilisés:
* **Istio's Destination Rules**
If you want to create multiple versions of your app, create a DestinationRule to define subsets that you can refer from the VirtualService.
* **Istio's Virtual Service**
Pour `manager ingress traffic` et `service-to-service communication`.
provide additional feature as:
  * external traffic routing/management 
    * Pod to external communication, HTTPS external communication
    * routing
    * url rewriting
* **Istio's Service Entry**
By default, all the external traffic in Istio is blocked. If you want to enable external traffic, you need to create a ServiceEntry to list what protocols and hosts are enabled for external traffic.

Configuration d'une application à monitorer
--
Istio est basé sur un proxy (envoy) interceptant tout le traffic d'un pod. Ce proxy doit être injecté directement à tous les pod d'un namespace.

1. **Configuration du namespace target** (où une injection d'un sidecare se fera automatiquement pour chaque pod créé)
```
kubectl label namespace default istio-injection=enabled 
```
2. **Configuration d'un gateway**. Il s'agit de configurer un composant `gateway` et d'un composant `virtual service` pour lier cette gateway à des ressources.
```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yam
```
/!\ Attention cette gateway doit pouvoir être accessible via une adresse IP (problème qui arrive généralement lorsque l'on installe une l'infrastructure K8 en bare-metal).
Il existe plusieurs solutions:
* external LB
* dans le cas de minikube, principe de [tunnel](https://medium.com/hackernoon/ingress-and-istio-gateway-resource-e5958846c26b)
3. **Traffic management**

*Routing*
* Définir les routes pour le versionning des applications**. Une `DestinationRule` par service (pod)
* Définir un virtual service permettant de répartir la charge entre différents services. (canary déployment par exemple) 

Possibilité de router en fonction d'un utilisateur, d'un header,...

*Circuit breaker*
Les circuit breaker se configurent à partir d'une 'Destination Rule'. Il s'agit de passer en mode `dégradé`avant d'avoir un problème.
eg.: un service ne supporte pas de connections concurrentes, une connection concurrente arrive. Au lieu de laisser entrer la requête le `circuit breaker`stoppera la requête avant d'invoquer le service.

*Mirroring*
Créer un Service virtuel qui permet non pas de router en fonction d'un pourcentage (canary dpyment) ou d'un header,... mais de dupliquer le trafic vers un autre service.

4. **Telemetrie**

*Metrics*
Métriques liées au système istio et au applications déployées: performance générale du systèmes, métriques liées au fonctionnement des différents composants de Istio, latence des requetes (99p, 95p, 50p,...)

Les métriques sont stockées dans `Prometheus`. Des dashboards sont déjà configurés dans `Grafana` pour visualiser les données.
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
```
[utilisation de Sysdig au lieu de Grafana](https://sysdig.com/blog/monitor-istio/)

*Gestion des logs*
* Communication entre `pods`:
Istio permet de définir des templates permettant de créer un log. On intercepte donc toutes les communications entre pod.
Le template doit se trouver dans le namespace d'Istio car il intérfere directement avec le composant `Mixer`.
Les ressources suivantes sont intéressantes pour créer un template:
  * experession language (Go Expression): [Istio / Expression Language](https://istio.io/docs/reference/config/policy-and-telemetry/expression-language/)
  * [Istio / Attribute Vocabulary](https://istio.io/docs/reference/config/policy-and-telemetry/attribute-vocabulary/)

/!\ Par défaut, Istio affiche les log en console, pour utiliser une pile du type ELK, il faut installer ES, Kibana et fluentd dans un namespace particulier. **-> cf mon script**

Quand la stack est installée, on peut visualiser les logs dans Kibana: 
```
kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601
```

* Intégrer les logs des conteneurs. Ce n'est pas une fonctionalité native mais on peut ajouter la fonctionnalité.
-> [Container and Service Mesh Logs - SE Notes by Alexey Novakov - Medium](https://medium.com/se-notes-by-alexey-novakov/container-and-service-mesh-logs-4382e481aa4b)

*Tracing*
Le tracing permet de visualiser l'enchainement des services lié à une requêtes.
L'outil `Jaeger`est disponible dans la stack Istio.
```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686
```
*Visualisation du service mesh*
L'outil `Kiali` est disponible dans la stack Istio pour permettre de visualiser et de modifier son service mesh (pas d'ajout, modification et check seulement sur les `viertual services` et les `destination rules`)

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001
```

Sécurité avec Istio
--

### RBAC

2 type de sécurité:
* **transport authentication** (Service2Service) -> serviceA peut contacter serviceB mais ne peut pas contacter ServiceC. Basé sur mTLS. Very needed for multi-tenant stack, aussi pour les attaques 'men in the middle'
* **origin authentication** (end_user2Service) request level authentication with Json Web Token en relation avec un fournisseur d'OpenID Connect ( ORY Hydra, Keycloak, [Auth0 + istio](https://auth0.com/blog/securing-kubernetes-clusters-with-istio-and-auth0/), [Firebase Auth](https://firebase.google.com/docs/auth/), [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect), and custom auth.)

2 façons de l'implémenter
* role binding: associer un role à une service (ServiceAccount)
* condition: être authentifier avec JWT (par exemple)

### Origin authentication

Utilisation du token JSON pour authentifier un utilisateur à un service.

bon article: [Getting Started With Istio - The Startup - Medium](https://medium.com/swlh/getting-started-with-istio-524628c025)


### end_user2Service (JWT)
TO CHECK, visiblement la demande d'un token est demandée pae l'adaptateur. Le l'app-web ne doit pas se soucier de demander le token -> à vérifier.
`App Identity and Access Adapter`
[Istio / App Identity and Access Adapter](https://istio.io/blog/2019/app-identity-and-access-adapter/)
 
Configuration d'un cluster K8s
--
Management ctrl est un control pane

Lors d'une configuration d'un multi-cluster il y a 2 profiles:
* remote: shared 'control pane' entre cluster
* multi-cluster-gateway: replicated ctrl pane
--> [Istio / Installation Configuration Profiles](https://istio.io/docs/setup/additional-setup/config-profiles/)


install
```
minikube start --cpus 4 --memory 10000

kubectl get namespace tiller  2> /dev/null || kubectl create namespace tiller


helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.3.0/charts/

```
