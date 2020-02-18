Concepts
==

1 Pod
--
Un pod est un hôte logique pouvant contenir plusieurs conteneurs. Ces conteneurs sont donc sur le même hote et partagent une adresse IP commune + un espace de port.

La mise à l'échelle est faite à partir du Pod.

Les conteneurs du pod peuvent communiquer entre eux, notamment avec `localhost`. Les conteneurs doivent donc coordonner leurs ports au sein d'un même pod.

Le pod spécifie également un emsemble de volumes partagés que les conteneurs peuvent accéder. Les données peuvent donc survivre au redémarrage d'un conteneur.

### 1.1 Namespace

Le namespace est un cluster virtuel hebergé sur le même cluster physique. Les namespace fournissent une base pour le naming. Le nom de ressources doit être unique dans un namespace. 

### 1.2 Kubelet

Est le contrôleur de pod. Kubelet est un démon qui fonctionne sur chaque noeud et prend en charge le démarrage, l'arrêt et la maintenance des pod.

les applications hautement disponibles, qui attendront que les pods soient remplacés avant leur arrêt et au moins avant leur suppression.

2 Resource Definition (déploiement de pod)
--
Les resources Kubernetes permettent de définir les composant d'une application (POD):
* service (service du POD)
* serviceAccount: processes /service tournant dans un pod (généralement 1)
* deployment: lie leservice avec une image (possibilité de réaliser des version différentes - a chaque déploiement, un pod est créé)

Il s'agit de fichiers `YAML` permettant de définir le déploiement d'une application/pod.

Ces définitions de ressources peuvent être étendues (notamment via des framework tournant sous K8 - Istio e.g.)


3 Exposition des services (type de services)
--
Kubernetes permet d'exposer les services de 3 manières différentes:
* **ClusterIP**: définit un service interne au cluster (application à l'intérieur du cluster peuvent y avoir accès, pas d'accès externe) - c'est le type par défaut.
* **NodePort**: ouvre un port spécifique sur tous les noeuds du cluster (VMs). Le port choisit doit être compris entre 30000 et 32767. Un service par port.
* **LoadBalancer**: permet d'associer une IP à un service. On peut y associer n'importe quel protocole.
-> cf externalip config

### 3.1 Ingress concept

Un Ingress n'est pas un type de service, l'ingress est un router qui se place en front des services. Il contrôle tout le traffic entrant. Istio est un controlleur d'ingress.

Belle illustration + plus d'[info](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

Un Ingress expose les routes HTTP et HTTPS de l’extérieur du cluster à services au sein du cluster

Un contrôleur d’Ingress est responsable de l’exécution de l’Ingress, généralement avec un load-balancer.

Le controleur d'ingress ne démarre pas avec le cluster, il est déployé dans un pod spécifique qui vérifie l'endpoint /ingresses.

Kubernetes maintient et supporte les controlleurs nginx et GCE ([Ingress Controllers - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/))

#### Ingress component in Istio

l'ingress controler d'Istio est divisé en 2 composant:
* Gateway: est utilisé pour configurer les istio-proxies (envoy). Une gateway permet à un type de traffic spécifique de rentrer dans le cluster via les proxys envoy.
* VirtualService: envoie le traffic à une ressource/service.

la commande suivante 
```
kubectl get svc istio-ingressgateway -n istio-system
80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32243/TCP,8060:31504/TCP,853:32380/TCP,15030:30112/TCP,15031:32472/TCP
```
permet de donner les redirection de port

plus d'info [info](https://medium.com/faun/istio-step-by-step-part-04-traffic-routing-path-of-istio-service-mesh-part-a-ingress-routing-28e03cdaa048)

#### autres Ingress component

il en existe d'autre comme
* traefik
* kong
* istio

[Ingress - Kubernetes](https://kubernetes.io/fr/docs/concepts/services-networking/ingress/)

##### traefic (reverse-proxy)

Pas de besoin de fichiers de configuration pour router les services.

Quand le service rémarre, on ajoute une information permettant à treafic de detecter que le service est bien up.

exemple docker compose:
```
whoami:
    # A container that exposes an API to show its IP address
    image: containous/whoami
    labels:
      - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"
```

### External LB concept

Lorsque l'on travaille sur une infrastructure bare-metal, il faut que K8 puisse exposer une adresse ip
* pour son ingress controller, 
* lors qu'un service est définit en load-balancing,
* ...

L'external load balancer est un service qui permet d'allouer une adresse ip an fonction d'une configuration réseau du type: `10.10.150.0/24`.

Installation de MetalLB sur:
* [Istio and MetalLB on Minikube - Emir Mujic - Medium](https://medium.com/@emirmujic/istio-and-metallb-on-minikube-242281b1134b)
* [Kubernetes Metal LB for On-Prem / BareMetal Cluster in 10 minutes](https://medium.com/@JockDaRock/kubernetes-metal-lb-for-on-prem-baremetal-cluster-in-10-minutes-c2eaeb3fe813)

Commandes
--

### Minikube

/!\ Minikube n'est pas packagé avec un external LB car Minikube n'est pas destiné à être utilisé sur plusieurs nodes. Work around:
```
minikube tunnel
```
va créer ce LB.
plus d'info: [LoadBalancer support with Minikube for Kubernetes](https://blog.codonomics.com/2019/02/loadbalancer-support-with-minikube-for-k8s.html)
#### installation

installe miniKube en 3 etapes sur mac : [Install Minikube - Kubernetes](https://kubernetes.io/docs/tasks/tools/install-minikube/#before-you-begin)

Upgrade minikube version: 
```
minikube version
brew cask install minikube
brew cask upgrade minikube
brew cask reinstall minikube
```

* hyperviseur (VirtualBox)
* kubectl
* Minikube

bonne resource: -> [Local Kubernetes setup on macOS with minikube on VirtualBox and local Docker registry · GitHub](https://gist.github.com/kevin-smets/b91a34cea662d0c523968472a81788f7)

#### concept et documentation
concepts Minikube: [concepts minikube](https://events.static.linuxfound.org/sites/events/files/slides/Managing%20Kubernetes%20with%20ManageIQ%20ContainerCon%20final.pdf)

#### Administation cluster (minikube)
```
minikube start
minikube start --memory=8000 --cpus=4 
minikube stop

Resetting and restarting your cluster
-> minikube delete

#interface web
minikube dashboard 

Afficher le service Nifi 
-> minikube service nifi

Afficher la liste des services
-> minikube service list
```

```
minikube ssh
$ top
GiB Mem : 12.4/15.7
```
This shows 12.4GiB used of an available 15.7 GiB RAM within the virtual machine.

### Kubectl (Administration K8)

ce fait avec le CLI `kubectl`
`kubectl` permet d'obtenir le résultat sour différente forme, comme en JSON (-o json).

```
#démarrer seulement un conteneur
kubectl apply -f ./nifi.yml
kubectl delete -f nifi/nifi.yml 

Afficher la liste des pods
-> kubectl get 
Obtenir seulement le nom d'un POD
-> kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}'
-> kubectl get pod --namespace logging

Afficher la liste des services pour un namespace
-> kubectl get svc -n istio-system
-> kubectl get service -n istio-system

Afficher la liste de namespaces
-> kubectl get namespace
-> kubectl delete namespaces <insert-some-namespace-name>

Supprimer tous les pod et service d'un namespace:
-> kubectl -n default delete po,svc,deployment --all
-> si on ne supprime pas les déploiements, 

Afficher la liste des gateways
-> kubectl get gateway

kubectl logs my-pod 
kubectl logs --tail=200 nifi-c8bf9844f-gqzjw 
kubectl logs -f nifi-6f7495757b-gffqh 
kubectl describe pods nifi-5d8477db68-tcg2z 

kubectl exec -it nifi-c8bf9844f-gqzjw -- sh

Event liés à kubernetes
-> kubectl get events --sort-by=.metadata.creationTimestamp

```
kubectl cheatsheet: [kubectl Cheat Sheet - Kubernetes](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
hubernetes aide: [Expose Pod Information to Containers Through Files - Kubernetes](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)



