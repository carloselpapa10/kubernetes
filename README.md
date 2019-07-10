# Kubernetes
A preparation for the Certified Kubernetes Application Developer (CKAD) program by [Cloud Native Computing Foundation](https://upload.wikimedia.org/wikipedia/en/0/00/Kubernetes_%28container_engine%29.png) (CNCF).

[CKAD Curriculum](https://github.com/cncf/curriculum/blob/master/CKAD_Curriculum_V1.14.1.pdf)

## Kubernetes Setup (Locally)

**Minikube** -> It runs a single-node Kubernetes cluster inside a Linux VM. It works on Windows, Linux and MacOS.
This [link](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#hyperkit-driver) will support you about how to set Minikube with Hyperkit as a virtualization software on MacOS.

[Install KubeCTL](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
**KubeCTL** is the platform using which you can pass commands to the cluster. So, it basically provides the CLI to run commands against the Kubernetes cluster with variuos ways to create and manage the Kubernetes components.

## Attend a Kubernetes free course by Google

This course is called [**Scalable Microservices With Kubernetes**](https://classroom.udacity.com/courses/ud615) and you will learn some Kubernetes basic concepts. The course is structured as follows:
  - Introduction to Microservices
  - Building containers with Docker
  - Kubernetes
  - Deploying Microservices

## Some basic concepts you should know

  - [Ngnix](https://www.nginx.com/) is a web server which can also be used as a reverse proxy, load balancer, mail proxy and HTTP cache.
  - Deployments keep our pods UP and RUNNING even the nodes they run on fail

```sh
$ kubectl run nginx --image=nginx:1.10.0
```
We can expose it outside of Kubernetes using the kubectl expose command.
```sh
$ kubectl expose deployments nginx --port 80 --type LoadBalancer
```

> Behind the scenes, Kubernetes created an external local balancer with a public IP address to attached it. Any client who hits up that public IP address will be routed to the pod behind the service. In this case it will be the nginx pod.

```sh
$ kubectl get services
```

Pods represent a Logical Application.
> Generally, if you have multiple containers with a hard dependency on each other, they would be packaged together inside of a single pod.

```sh
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: carloselpapa10/monolith:10.1.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```
```sh
$ kubectl create -f pods/monolith.yml
```

Use the kubectl describe command to get more information about the monolith pod.
```sh
$ kubectl describe pods monolith
```
> **Note:** Pod allocated a private IP address by default and can´t be reached outside of the cluster.
> So, use kubectl port-forward command to map a local port to a port inside the monolith pod.

```sh
$ kubectl port-forward monolith 10080:80
```

Now, we can hit that pod, using:
```sh
$ curl http://127.0.0.1:10080
```

We can check the log of a specific container by using the kubectl log command.
```sh
$ kubectl logs monolith -f
```

We can use the kubectl exec command to run an interactive shell inside the monolith pod.
```sh
$ kubectl exec monolith --stdin --tty -c monolith
/bin/sh
# ping -c 3 google.com
# exit
```
> Note that we can test external connectivity using the ping command.

Sometimes a container on a pod can be Up and Running but an application inside of the container mught be malfunctioning. Kubernetes uses **health** and **readiness** checks.

**Readiness probes** indicate when a pod is ready to serve traffic. If a readiness check fails then the container will be market as not ready and will be removed from any load balancer.

**Liveness probes** indicate a container is alive. If a liveness probe fails multiple times, then the container will be restarted.

> **Kubelet** is responsible for making sure that a pod is healthy

Secrets and ConfigMaps

ConfigMaps and Secrets are similar except ConfigMaps don't have to be sensitive data. ConfigMaps use environment variables.

What is a reverse proxy server?

In computer networks, a reverse proxy is a type of proxy server that retrieves resources on behalf of a client from one or more servers. These resources are then returned to the client, appearing as if they originated from the proxy server itself.

> Using reverse proxy, the client does not know which server is connecting to

Example: Demo & Nginx (Github repo)

ConfigMap
```sh
$ kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
```

Deployment
```sh
$ kubectl create -f pods/demo.yml
```

Port-forward
```sh
$ kubectl port-forward demo 10443:80
```

Now, you can use another console to hit the service.
```sh
$ curl http://127.0.0.1:10443
```

> Note that the kubectl port-forward command is great for testing pods directly, but in production, you want to expose pods using services.

To list all deployments:
```sh
$ kubectl get deployments --all-namespeces
```

Then to delete the deployment
```sh
$ kubectl delete deployments -n NAMESPACE deployment DEPLOYMENT
```

Kubernetes pod get recreated when deleted
You need to delete the deployment, which should in turn delete the pods and replicasets.

Services (Stable endpoints for pods)

Persitence Endpoint for pods.
 - Use labels to select pods.

The level of access that a service provides to a set of pods depends on the service type:
 - clusterIp (internal IP only)
 - NodePort (it gives each node an external IP)
 - LoadBalancer
 - ExternalName

Kubernetes Service pod.
```sh
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  selector:
    app: "demo"
    secure: "enabled"
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 31000
  type: NodePort
```

```sh
$ kubectl create -f services/demo-svc.yml
```

> Note that the range of valid ports by default is 30000-32767 

Using of labels

In a certain time, a service could not match with any endpoints. One way to troubleshoot this issue is to use the kubectl get pods command with a label.

```sh
$ kubectl get pods -l "app=demo"
$ kubectl get pods -l "app=demo,secure=enabled"
$ kubectl describe pods secure-demo | grep Labels
# add a label to a pod
$ kubectl label pods secure-demo "secure=enabled"
# check endpoitns in a service
$ kubectl describe services demo | grep Endpoints
```

> Note: Get the public IP address of one of your nodes that is running a demo pod. If your are using Minikube -> $ kubectl cluster-info then $ curl http://<public-node-ip>:<node-port>

Type NodePort

The kubernetes control plane allocates a port from a range specified by --service-node-port-range falg (default: 30000 - 32767). Each node proxies that port (the same port number on every node) into your service.
