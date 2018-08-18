# k8s-ansible
Deploy container from private repository with minikube

## Installation de minikube avec un registre docker privé

#### Installationd du registre docker

Il faut une machine virtuelle (vagrant ou KVM) et ensuite lancer le playbook suivant :

[https://github.com/jreg2k/ansible-docker-registry](https://github.com/jreg2k/ansible-docker-registry)


### Installation de minikube

Suivre l'installation standard (j'ai installé' minikube avec le driver kvm2)

[https://kubernetes.io/docs/tasks/tools/install-minikube/](https://kubernetes.io/docs/tasks/tools/install-minikube/)


#### Configurer les secrets pour pouvoir déployer des containers depuis kubernetes [^1] [^2] [^3] :


[^1]: [https://medium.com/@alombarte/setting-up-a-local-kubernetes-cluster-with-insecure-registries-f5aaa34851ae](https://medium.com/@alombarte/setting-up-a-local-kubernetes-cluster-with-insecure-registries-f5aaa34851ae)
[^2]: [https://github.com/kubernetes/minikube/blob/master/docs/insecure_registry.md](https://github.com/kubernetes/minikube/blob/master/docs/insecure_registry.md)
[^3]: [https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

On utilisera l'utilisateur créé dans le playbook ansible de déploiement du registre docker.

```
kubectl create secret docker-registry regcred --docker-server=docker-registry.localdomain --docker-username=james --docker-password=james --docker-email=james@localdomain
```

Vous devez lancer minikube avec le paramètres --insecure-registry="docker-registry.localdomain"

```
# minikube start --insecure-registry="docker-registry.localdomain" --vm-driver="kvm2"

```

Si vous avez (ne serait-ce qu'une fois) déjà lancer minikube sans le paramètre

**--insecure-registry** vous devez supprimer l'instance en cours :

```
minikube delete
```
pour en déployer une nouvelle avec le paramètre **--insecure-registry**.


Une fois minikube démarré, vous devez activer les addons suivants :

```
minikube addons configure registry-creds
minikube addons enable registry-creds
```

Tester la connexion avec :

```
minikube ssh
docker login docker-registry
docker pull docker-registry/my-application-container
```


## Créer le container

```
 cd Minikube/nodejs
 sudo docker build -t hello-node .
 sudo docker tag 0bff5a8d5af4 docker-registry.localdomain/hello-node:v3
 sudo docker push docker-registry.localdomain/hello-node:v3
```


##  Déployer un pod contenant l'application "containerisée"

avec regcred : nom du secret créé précédement.


```
apiVersion: v1
kind: Pod
metadata:
 name: private-reg
spec:
 containers:
 - name: private-reg-container
   image: <your-private-image>
 imagePullSecrets:
 - name: regcred

```

 **Pod**:  (ensemble des containers nécesaires à l'application)
 **Deployment**: (comment sera déployée cette application - ce Pod: ha, seuil, etc..)
 **Service**: (expostion du déployment : kubectl expose deployment nom_du_deployment --type=LoadBalancer)
 on bind le nom DNS sur l'adresse fournit pas la commande "kubectl expose"


fichier pour déployer une application avec deux réplicats :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-node
spec:
  selector:
    matchLabels:
      run: hello-node
  replicas: 2
  template:
    metadata:
      labels:
        run: hello-node
    spec:
      containers:
      - name: "hello-node"
        image: "docker-registry.localdomain/hello-node:v3"
        ports:
        - containerPort: 8080
```
[https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-deployment](https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-deployment)

1. faire l'application
2. créer le container (cf Dockerfile)
3. créer un "deployment"
4. creer un service
5. mise-à-jour de l'application : avec l'édition du deployment et le changement de version

## Avec Ansible et le module k8s (pour une application simple : avec un container et sans docker-compose)

```
Ansible/
├── gcp_k8s.yml
├── hosts
├── k8s.sh
└── roles
    └── first-test
        ├── files
        │   ├── deployment.yml
        │   └── service.yml
        ├── tasks
        │   └── main.yml
        ├── templates
        │   ├── deployment.yml
        │   └── service.yml
        └── vars
            └── main.yml
```

hosts:

```
[minikube]
127.0.0.1
```

gcp_k8s.yml

```
---

- hosts: minikube
  roles:
    - first-test

```


k8s.sh

```
#!/bin/env bash

ansible-playbook -i hosts gcp_k8s.yml -vvv
```


roles/first-test/tasks/main.yml

```
---

- name: Get a list of all service objects
  k8s:
    host: "https://192.168.39.172:8443"
    key_file: "/home/james/.minikube/client.key"
    verify_ssl : no
    kubeconfig: "/home/james/.kube/config"
    api_version: v1
    name: testing
    api_version: v1
    kind: Namespace
    state: absent
  register: service_list
  ```

**host** : Kubernetes API endpoint

**key_file** :  client-key in  ~/.kube/config (kubectl config view)

**kubeconfig** : kubernetes config file


### Contenu du playbook

roles/first-test/templates/deployment.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ application_name }}
  namespace: {{ namespace }}
spec:
  selector:
    matchLabels:
      run: {{ application_name }}
  replicas: {{ desired_replicas }}
  template:
    metadata:
      labels:
        run: {{ application_name }}
    spec:
      containers:
      - name: "{{ application_name }}"
        image: "{{ registry }}/{{ application_name }}:{{ application_version}}"
        ports:
        - containerPort: {{ container_port }}
```

roles/first-test/templates/service.yml :

```
apiVersion: v1
kind: Service
metadata:
  name: {{ service_name }}
  namespace: {{ namespace }}
  labels:
    run: {{ application_name }}
spec:
  externalTrafficPolicy: Cluster
  selector:
    run: {{ application_name }}
  type: LoadBalancer
  sessionAffinity: None
  ports:
    - name: {{ service_name }}
      nodePort: {{ node_port }}
      port: {{ port }}
      protocol: TCP
      targetPort: {{ target_port }}
status:
  loadBalancer: {}

```

Pour accder au services de type *LoadBalancer* depuis minikube, le commande

**minikube service hello-node-service**

devra être utilisée.
