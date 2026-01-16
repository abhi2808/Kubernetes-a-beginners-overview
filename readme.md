# containers

isolated env, sharing same OS kernels, type: LXC, LXD, LXCFS, docker historically used LXC now containered+runc. docker provides a high level tool to use them.

OS => kernel(interact with HW)+software(what diffs generally)

docker can run any os as long as its based on same kernel. When you run a linux container on windows, its actuall running on a linux vm. 

Thus over VM, containers are lightweight, faster to boot, but lack proper isolation
generally used together => container on VM host

We can store the images(package templates, runs same on any host) on a container registery which can be used to retrive them and then spin up the containers(running instances of images).

solve: compatibility issue + environment set up time

container orcehstrators are responsible for managing the number of and connetivity between containers. Eg docker swamp, K8s etc.


# Architecture / baisc concepts

- nodes: machine physical or virtual, with K8s installed and where containers(pods) would be launched.

- cluster: group of nodes to share load, managed by master-node(responsible for orchestration)

master-> api server(to communicate), controllers, etcd(stores info eg health),  scheduler

worker(minion)-> kubelet(agent-manages pods)(to talk), container runtime(docker, crio, rkt), kube-proxy

- kubectl: tool to manage k8s cluster


# Container runtimes

"CRI"(container runtime interface) allows any vendor to be a container runtime for K8 but docker doesn't support it and so runs without it(due to popularity implemented seperately as dockershim). Now no longer supported. Though containerd(the demon)(now as a part seperate from docker) seperately is supported. ctr is used to debug containerd. For running commands similar to docker we use "nerdctl".

"crictl" provides command line functioanlity for cri supported runtimes generally for debugging and inspecting.


# Setup

k8s service available as: loacal(minikube - preconfigured single node k8s cluster, kind), cloud managed(EKS, AKS), playground(KodeKloud).

![alt text](image.png)


# Pods

containers not deployed directly, are encapsulated into a k8s object called pod they are the single instance of an application.

generally 1 container of a type per pod the other containers are helpers(if scale use multiple pods)(on a pod get created and die together)(share same network ie can communicate + share same storage space)

kubectl run nginx(pod name) --image=nginx

kubectl describe pod nginx(pod name), to get info about it


# YAML

for configuration data,

key-value pair-> 

fruit: apple
liquid: water

arrays/lists-> ordered

fruits:
-   orange
-   apple
-   banana

dictionary/maps-> unordered, the nesting is managed by spaces

banana:
  calorie: 105
  fat: 0.4
grapes:
  calorie: 62
  fat: 0.3

complex example, list of dicts:

fruits:
-   banana:
      calorie: 105
      fat: 0.4
-   grapes:
      calorie: 62
      fat: 0.3    


# YAML in kubernetes

Pods:

(root level properties/required fields)

apiVersion: (pods=v1, service=v1, replicaset=apps/v1, deployment=apps/v1)
kind: (type of obj)
metadata: (name(string), labels(dict(app, type etc))(siblings) etc)(in matadata only what k8s considers metadata, but labels => your wish)

spec: (object info)(containers(list->container per element) containing dict(name, image) as elements)

kubectl create/apply -f pod-defination.yml

![alt text](image-1.png)


# replication controller 
helps run multiple instances of a single pod (or get one up if current pod goes down) providing high avability, load-balancing and scaling. Now this has been replaced by "replicaSet"

->all nest defination files

- replication controller:
apiVersion: v1
kind: ReplicationController
metadata: (name(string), labels(dict(app, type etc))(siblings) etc)(in matadata only what k8s considers metadata, but labels => your wish)

spec: (object info)(template: (pod template ie the content of pod.yml exept apiVersion and kind))(replicas: number required)

ps: selector is also present here but by default takes the value of labels provided in the pod defination files.

- replicaSet:
apiVersion: apps/v1
kind: ReplicaSet
metadata: (name(string), labels(dict(app, type etc))(siblings) etc)(in matadata only what k8s considers metadata, but labels => your wish)

spec: (object info)(template: (pod template ie the content of pod.yml exept apiVersion and kind))(replicas: number required)(selector: helps replica set identify what pods fall under it, as can manage pods existing before the replica set creation (matchLabels: key-pair to match with the metadata of the pods))


they provide a way to monitor the pods and know which pods to minitor using labels.

for replica-set even if the req no of pods exist and the template is currently not needed, we still need to provide it as we need to spin up new containers if one of the pre exiusting goes down

to increase(scale) the nuber of pods:
- modify the yml file
- kubectl scale --replicas=6 -f replicaset-definition.yml(wont affect the number in the file)
- kubectl edit replicaset myapp(set name), opens the running config of the set, applied as soon as saved
- also can be increased based on load(would be discussed later)

![alt text](image-2.png)

if you delete a pod new one is spun up, if more than required are created(manually) the extras are terminated


# Deployments

provide ability to upgrade instances by rolling updates, undo changes and resume changes

basic structure same as replicaSets->

apiVersion: apps/v1
kind: Deployment
metadata: (name(string), labels(dict(app, type etc))(siblings) etc)(in matadata only what k8s considers metadata, but labels => your wish)

spec: (object info)(template: (pod template ie the content of pod.yml exept apiVersion and kind))(replicas: number required)(selector: helps replica set identify what pods fall under it, as can manage pods existing before the replica set creation (matchLabels: key-pair to match with the metadata of the pods))

## rollout and updates

when we create a deployment it triggers a new rollout which creates a new replica set recorded as a new deployment revision. when the container is upgraded a new deployment revision is created(revision 2). This helps us track the version of deployment and rollback if necessary!! 

kubectl rollout status deployment/myapp-deployment(deployment name)

kubectl rollout history deployment/myapp-deployment (deployment name) (gives revision+history) (to track teh deployement add the --record tag while creating/applying the deployment)


how do you upgrade the replicas of your instance:

- destroy all then recreate(issue: downtime, not used)

- kubectl set image deployment/httpd-frontend httpd=httpd:2.4(would cause mis match with config file)

- slowly one-by-one, rolling updates(default assumed just apply) when you create  a deploment a replica set is created automayically, when you are updating a new replica set is created as the pods in the newer RS are created as the ones in the older RS are deleted

to rollback: kubectl rollout undo deployment deployment-name (cant do if the older replica sets have been deleted)


# Networking

Each pod gets its own internal Pod IP. Pods are attached to a virtual network created by the CNI plugin, not directly to the node IP (example Pod CIDR 10.244.0.0/16, so Pod IPs like 10.244.0.2, 10.244.0.3, etc). Pod IP changes when pods are recreated.

Kubernetes expects networking to be set up such that all pods can communicate with each other across nodes without NAT (using CNI plugins like Flannel, Calico, Cilium, NSX, Cisco), which creates a cluster-wide network with unique IPs and routing, allowing pod-to-pod communication inside the cluster (pod → CNI → pod).

On top of this network, Kubernetes provides Services with ClusterIP, which gives a stable internal virtual IP and DNS name for accessing a group of pods and load-balancing traffic between them. Pods inside the same cluster usually communicate using the Service ClusterIP instead of direct Pod IPs because Pod IPs change, while ClusterIP remains stable (pod → ClusterIP → CNI → pod). ClusterIP is accessible only inside the cluster and is not exposed to external users.

# Services

enable communication bw components within and outside of the application.

connections: 

1) users->app

- by sshing into node(not ideal)
- service acts as a middle men, listen to port on the node and forward req on that port to a port on the pod running the application this is  "nodePort(the port on node) service" teh target port is on the pod

apiVerison: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  ports: 
   - targetPort: 80(on pod)(default: same as port)
     port: 80(on service server has a cluster ip)(mandatory)
     nodePort: 30008(port on node)(30000-32767)
  selector:
    app: my-app

many pods: no problem as matched using labels
pods across nodes: services spans across all nodes, so can access app using ip of any node in cluster but same port-number available

![alt text](image-3.png)
cluster ip is created for the service automatically


2) fend node->bend node

cluster ip creates a virtual ip inside the cluster to allow communication bw grou fend and bend servers. Allow a accessibility as a group( effective for microservice architecture)

apiVerison: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP(cluster ip is the default)
  ports:
    - targetPort: 80 (where backend is exposed)
      port: 80 (where service is exposed)
  selector:
    app: my-app

3) node to external service

loadbalancer service, to employ external load balancing from a supported cloud provider

apiVerison: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
    - targetPort: 80 
      port: 80
  selector:
    app: my-app


# Microservices

voiting app eg -> voting-app, redis, worker, mysql, results-app

using link flag to connect docker containers is deprecated, use diff approach

to expose one service to another asip changes we need a service(cluster IP)->2-> redis and postgres

1 nodeport services each for voting and result app too 

demo:

kubectl apply -f . (to apply for entire directory)

![alt text](image-4.png)

![alt text](image-5.png)

kubectl get node -o wide

GPT{
minikube service vote --url, creates a dynamic proxy tunnel from your Windows host's localhost (127.0.0.1:61968) to the NodePort inside the Minikube Docker container, bypassing networking limitations. Direct access to http://192.168.49.2:31000 fails because the Docker driver runs Minikube as a container without automatic host port publishing or NAT forwarding for high NodePorts on Windows.
​

Docker Driver Limitation
The Docker driver lacks VM-style networking, so NodePorts (30000-32767) on the container IP (192.168.49.2) aren't reachable from the host without manual port mapping or SSH tunneling. Minikube's service command handles this by running a persistent proxy process in the foreground terminal.
​}

![alt text](image-6.png)

![alt text](image-7.png)


i have implemengted this using 2 approaches i.e. using only pods and using deployments, remember just scaling the number of replicas for db may cause consistency issues, to prevent use persitent volumes.


in ideal scenario use{

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi

in worker node add->
env:  # ADDED: DB connection for INSERT success
        - name: POSTGRES_DB
          value: vote
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres

}

while testing vote using incognito tabs or diff browsers, as the code is configured to evaluate session cookies, if voting is done using diff tabs of 1 browser shows only 1 vote!!(almost lost my mid over this)

# kubernetes on cloud

k8s-> 
1) self hosted/turnkey-> provision, configure, maintain and write scripts on the VM yourself. kops and kubeone in aws.

2) hosted/managed->  k8s as a service, providers does everything(managed master nodes). eg GKE