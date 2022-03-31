# SuzieQ on k8s

## This readme is obsolete, it needs to be updated ##

- split gui, poller and rest into separate pods
- use env variables where needed
- poller configured as a statefulset
- use ingress to expose suzieq webui and rest server
- updated configuration files to match current suzieq requirements and showing multiple namespaces
- use official suzieq image

## Why?

I recently discovered [SuzieQ](https://github.com/netenglabs/suzieq), it's a wonderful new tool that Dinesh Dutt and Justin Pietsch have been working on.  For more details on what exactly SuzieQ is see the intro [post](https://elegantnetwork.github.io/posts/Suzieq/) by Justin & Dinesh.

As to why put it on k8s?  I don't know k8s well, it's just the compute hammer I have to work with in my current environment. That said, I do happen to like k8s as a hammer, and luckily suzieq seems to do a pretty good job of being a k8s nail.  
 
This document started as a weekend project to walk thru getting suzieq on k8s in my private lab for no other reason than it seemed fun.  The more I worked with it on k8s the more I liked the idea, and the more I liked the idea of deploying in my real physical lab.
 
The environment I have for my network management software is a managed k8s environment.  It's what I have access to for "managed compute". It also has access to the management plane of my network gear.  This means if I want to go to the real lab or prod, I have a vested interest in being able to deploy suzieq on k8s.
 
The idea of using a HTTP based API for accessing much of the info that suzieq collects appeals to me.  I like the idea of connecting to a suzieQ deployment versus running the pollers directly on my workstation.  Playing with that as a puzzle problem appeals to me as much as suzieQ as a product appeals to me as a network engineer.    I like the idea of the journey and the destination as it were.
 
## Who is this for, why share it
 
Mostly me, this started as notes for me to reference later on how to do it, but I don't imagine I'm gonna be the first person that thinks of running SuzieQ on k8s.  .
 
## The basic target
 
For just playing around in the lab I want to start out with something that sticks as close as I can to the docker example that is in the suzieq repository.  In that example there are two containers; one for sq-poller, and another for the suzieq-cli.   I'm going to replicate the same in a k8s deployment set with replicas set to 1.
 
My goal will be to run run 3 containers in a single k8s pod, one container for each of the following processes that make up suzieq.  
 
1. `sq-poller`
2. `sq-rest-server.py`
3. `suzieq-gui`

For simplicity I will use the same docker container as the docker demo

What's needed
 
- For persisting data I will use a k8s [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- For managing config I will store configs as a k8s [config map](https://kubernetes.io/docs/concepts/configuration/configmap/)
- For storing secrets like certificates, keys or passwords I will use k8s [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- To expose the Rest API and GUI outside of the cluster, I will use a k8s [service](https://kubernetes.io/docs/concepts/services-networking/service/)
- To tie it all together I will use a k8s [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
 

The assorted yml files referenced in this post as well as the content of this post is available on [github](https://github.com/hyposcaler/suzieq-on-k8s/)

## Generating the certificates
 
I need a cert for the demo deployment, a self signed cert will do.  In production the certificate would be signed by my internal CA.  I can use openssl to generate the certificate with the following command 
 
`openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes`
 
Below is an example of using openssl in interactive mode to generate a selfsigned certificate.
 
```
[dev-suzieq]$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
Generating a 4096 bit RSA private key
....................++
........................................++
writing new private key to 'key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:NC
Locality Name (eg, city) [Default City]:Charlotte    
Organization Name (eg, company) [Default Company Ltd]:Hyperbolic Hyposcaler Industries
Organizational Unit Name (eg, section) []:network
Common Name (eg, your name or your server's hostname) []:suzieq.network.hyposcaler.net
Email Address []:devnull@hyposcaler.io
```
 
After running the command and filling in the details it should produce two files: `key.pem` and `cert.pem`.   
 
Once I have the certs I'll store them as secrets on the k8s server.  For the lab I'll just use kubectl for that.  The following command will create the secret from the cert.pem and key.pem created by openssl earlier.  
 
 
```
kubectl create secret tls suzieq-tls-secret \
  --cert=cert.pem \
  --key=key.pem
```
 
Regardless of how the cert is created, a k8s TLS secret is typically the interface for the deployment to access it.  There are a variety of methods to manage secrets securely on k8, they generally tend to have use of k8s secrets in common as the delivery method for exposing the secrets to containers.  
 
In practice while I may not be doing exactly what I would in prod, the way I'm doing it creates a good approximation of what suzieq would see in prodution in my environment.
 
Once I have created the k8s TLS secret, the key and cert files from openssl can be deleted.  
 
## Generating the configMaps
 
There are two config related files that suzieQ uses
 
The first is `suzieq-cfg.yml`. It is the main suzieq config file.  The suzieq config holds pointers to the assorted paths for files it uses.  It also holds references to the the API_KEY used by the Rest server for auth.  Lastly it also has pointers to the locations of the TLS certificates. 
 
The following is a typical `suzieq-cfg.yml` file
 
```yaml
data-directory: /suzieq/parquet
service-directory: /suzieq/config
schema-directory: /suzieq/config/schema
temp-directory: /tmp/
# kafka-servers: localhost:9093
logging-level: WARNING
period: 15
API_KEY: 496157e6e869ef7f3d6ecb24a6f6d847b224ee4f
rest_certfile: /suzieq/tls/cert.pem
rest_keyfile: /suzieq/tls/key.pem
```
 
In fact it's the default config, with one small change:  I have added a subdirectory under /suzieq/ to hold the certificates.  I like having an empty directory to mount the TLS secrets into.
 
The other config we need to keep track of is the inventory.  The inventory is also a yaml based file. 
 
I use this as my standin for inventory
 
```yaml
- namespace: eos
  hosts:
    - url: https://neteng:arista123@10.255.0.10 devtype=eos
```
 
Note I'm storing the passwords in the clear here, for a small private virtual lab that gets spun up and down as needed, I'm happy to do that.  This is not something I would want to do in production.  It also happens that in my specific production instance I need a way to support password based auth.   I would ultimately prefer to store the credential as a k8s secret, and pass that secret into the container via an environment variable, or file.
 
If I were using SSH keys, I would use a k8s ssh key secret and mount them into the suzieq directory under a keys subfolder, similar to what I do do with TLS.  You specify a path to the key in the suzieq poller config, so it would work well for k8s SSH secret.
 
For the suzieq-cfg.yml a config map fits the bill nicely.  For lab purposes or small scale deployments, a configMap should work for inventory as well, there's an upper limit of 1MB, but that's a lot of config.
 
Either way I mash these up into the same yaml file named configmap.yml.  
 
The resulting file looks like the following.
 
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: suzieq-inventory
  namespace: suzieq
data:
 inventory.yml: |
    - namespace: eos
      hosts:
        - url: https://neteng:arista123@10.255.0.10 devtype=eos
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: suzieq-config
  namespace: suzieq
data:
  suzieq-cfg.yml: |
    data-directory: /suzieq/parquet
    service-directory: /suzieq/config
    schema-directory: /suzieq/config/schema
    temp-directory: /tmp/
    # kafka-servers: localhost:9093
    logging-level: WARNING
    period: 15
    API_KEY: 496157e6e869ef7f3d6ecb24a6f6d847b224ee4f
    rest_certfile: /suzieq/tls/tls.crt
    rest_keyfile: /suzieq/tls/tls.key
```
 
then use kubectl to create the two configmaps on the k8s cluster by running `kubectl create -f samples/k8s/configmap.yml` from the root of the repo
 
once created kubectl can be used to to verify they exist:
 
```cli
[dev-suzieq]$ kubectl -n suzieq describe configmap
NAME               DATA   AGE
suzieq-config      1      25h
suzieq-inventory   2      20h
[dev-suzieq]$
```
 
After they are created, they can later be edited in place, or overwritten with new config maps.
 
## Persistent Volumes
 
For the persistent volume, create a file named pvc.yml with the following contents
 
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: suzieq
  name: parquet-pv-claim
spec:
  accessModes:
    - ReadWriteOnce     # will only have 1 pod so read 
  resources:            # write once for now is fine
    requests:
      storage: 5Gi      # 5Gi completely arbitrary value. 
```
 
deploy it to the cluster via `kubectl create -f samples/k8s/pvc.yml`, this will cause the cluster to set aside 5Gi of storage, it's lifecycle is not tied to the pod.  The pods are ephemeral, the storage can persist across the life of many pods. 
 
## the deployment
 
with the volumes, secrets, and storage lined up on the k8s cluster, all that remains is the deployment for the pod.
 
use a k8s [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to deploy suzieq.
 
The documentation does a pretty good job of explaining what a deployment is. Refer to the k8s deployment documentation for more details. 
 
This deployment will ensure that there are always 3 containers running.  The containers only really have 3 differences between them
 
- the command executed at start
- the volumes they have access to
- the resource memory and cpu requests/limits
 
### command executed
 
There 3 containers, one for each of the the 3 processes/apps that make up suzie-q.  
 
1. sq-poller
2. sq-rest-server.py
3. suzieq-gui
 
 
sq-rest-server.py and suzieq-gui are pretty straight forward, in this deployment they don't need any additional args so their "commands" are going to be `command: ['sq-rest-server.py']` and `command: ['suzieq-gui']` respectively.
 
For poller we need to pass in some additional args 
 
-k to disable host key checking, and then an argument for the inventory
 
The final command looks like this 
 
`command: ['sq-poller', '-k', '-D', '/suzieq/inventory/inventory.yml']`
 
 
Note: I don't know that I need to use two different files here, but I seem to recall having had issues with doing two namespaces in the same inventory file.  I don't mind the need for two files, but would prefer being able to point the poller at a directory if multiple files are required for multiple namespaces.
 
### volumes they have access to
 
I gave all containers access to all volumes except for the and poller config.  I don't really know what makes sense here, my instinct was to just keep the poller config off of the two containers that are external facing.
 
### the resource memory and cpu requests/limits
 
I don't have a good idea what makes sense for these, I picked the values mostly just to try and keep the resources for the pod under what I know my nodes in my private k8s clusters have (t3.mediums).
 
Until I get into the real lab I won't really have a feel for what practical values here might be.
 
```yaml
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "500Mi"
      cpu: "500m"   
```
 
 
 
 
## Exposing the Services 
 
A simple service to front for the gui and rest API can be created via k8s loadbalancer type service.
 
service.yaml contains the following 
 
 
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: suzieq
  annotations:
    # this annotation isn't generally needed, 
    # but my private lab is build on AWS, and I don't want a 
    # public facing VIP
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  labels:
    app: suzieq
spec:
  ports:
  - port: 8000
    name: sq-rest
  - port: 8501
    name: sq-gui
  type: LoadBalancer
  selector:
    app: suzieq  # this is the selector the service 
                 # expects to find on containers it
                 # it is exposing. 
```
 
Create the service by running the command `kbuectl create -f samples/k8s/service.yml` from the root of the repo
 
You can validate the service via `kubectl -n suzieq get service`
 
```
[dev-suzieq]$ kubectl -n suzieq get service
NAME     TYPE           CLUSTER-IP       EXTERNAL-IP                                                                        PORT(S)                         AGE
suzieq   LoadBalancer   192.168.72.180   internal-a7839d0d9fb6bbc89b7e44b26f52254c-1179302016.us-east-1.elb.amazonaws.com   8000:30411/TCP,8501:30344/TCP   46m
[dev-suzieq]$ 
```
 
We don't need to have the systems in place it points to yet.  The service will happily exist without actual containers to work with yet.  It will simply wait for containers where the "selector" on the service matches the "selector" on the container. 
 
## The whole deployment file 
 
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: suzieq
  name: suzieq
  labels:
    app: suzieq
spec:
  replicas: 1  # We only need one pod here          
  selector:
    matchLabels:
      app: suzieq          
  template:
    metadata:
      labels:
        app: suzieq
    spec:
      volumes:                # here we name each of 
          - name: suzieq-tls  # of the volumes to be 
            secret:           # referenced in the containers
              secretName: suzieq-tls-secret
          - name: parquet-volume
            persistentVolumeClaim:
              claimName: parquet-pv-claim
          - name: config-volume
            configMap:
              name: suzieq-config
          - name: inventory-volume
            configMap:
              name: suzieq-inventory
      containers:             
      - name: sq-rest-server            # the rest-server container
        image: ddutt/suzieq-demo
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:                     # a list of one or more 
          - name: suzieq-tls              # of the previously named
            mountPath: "/suzieq/tls/"     # volumes to be mounted 
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml 
            subPath: suzieq-cfg.yml 
        command: ['sq-rest-server.py'] # our "entry point"
        
      - name: sq-poller               # the sq-poller container
        image: ddutt/suzieq-demo
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml
            subPath: suzieq-cfg.yml
          - name: inventory-volume
            mountPath: /suzieq/inventory/inventory.yml
            subPath: inventory.yml
        command: ['sq-poller', '-k', '-D', '/suzieq/inventory/inventory.yml']
 
      - name: suzieq-gui            # third and final container for the gui
        image: ddutt/suzieq-demo
        resources:
          requests:
            memory: "128Mi"
            cpu: "125m"
          limits:
            memory: "250Mi"
            cpu: "250m"        
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: suzieq-tls
            mountPath: "/suzieq/tls/"
            readOnly: true
          - name: parquet-volume
            mountPath: "/suzieq/parquet"
          - name: config-volume
            mountPath: /root/.suzieq/suzieq-cfg.yml
            subPath: suzieq-cfg.yml          
        command: ['suzieq-gui']
      # imagePullSecrets:
      # - name: regcred
```
 
Can deploy it with `kubectl apply -f samples/k8s/deployment.yml`
 
If everything is working correctly we should be able to use `kubectl -n suzieq get all` and see 3/3 containers running.

```
[dev-suzieq]$ kubectl -n suzieq get all 
NAME                          READY   STATUS    RESTARTS   AGE
pod/suzieq-66dc944496-vx7tj   3/3     Running   0          42h
 
NAME             TYPE           CLUSTER-IP        EXTERNAL-IP                                                                        PORT(S)                         AGE
service/suzieq   LoadBalancer   192.168.174.140   internal-a7839d0d9fb6bbc89b7e44b26f52254c-1179302016.us-east-1.elb.amazonaws.com   8000:31552/TCP,8501:31112/TCP   43h
 
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/suzieq   1/1     1            1           42h
 
NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/suzieq-66dc944496   1         1         1       42h
[dev-suzieq]$
```
 
We can test hitting the rest endpoint via the LB with curl
 
```
[dev-suzieq]$ curl --insecure 'https://internal-a7839d0d9fb6bbc89b7e44b26f52254c-1179302016.us-east-1.elb.amazonaws.com:8000/api/v1/interface/show?&access_token=496157e6e869ef7f3d6ecb24a6f6d847b224ee4f'
[{"namespace":"eos","hostname":"lab1-fab1-pod1-leaf1","ifname":"Ethernet100","state":"up","adminState":"up","type":"ethernet","mtu":1500,"vlan":0,"master":"","ipAddressList":[],"ip6AddressList":[],"timestamp":1611108293987},{"namespace":"eos","hostname":"lab1-fab1-pod1-leaf1","ifname":"Ethernet1","state":"up","adminState":"up","type":"ethernet","mtu":1500,"vlan":0,"master":"","ipAddressList":["10.255.0.10/24"],"ip6AddressList":[],"timestamp":1611108293987}]
[dev-suzieq]$ 
```
 
For production I may not use a LoabBalancer type service specifically, but would likely still expose the ports on the pod to the rest the world via a k8s service of some type.
 
## What next
 
This isn't quite production ready.  The objective here was to recreate the docker example for suzieq but on k8s.   I wanted to use that as a starting point for getting k8s into my real lab, and ultimately production as well.   In that context; success has occurred.     
 
My next steps are to figure out how to scale it out.   A single pod with a single sq-poller instance will suit for some basic exploration in the lab.  At some point there will be a need for more than one sq-poller, that also means a need to handle splitting inventory across what will likely be multiple pods.  How do I automate that, how do I manage that etc. etc.  
 
Before that I need to get a feel for what the limits of a single simple pod are, and what the ends and outs of managing the parquet files will be.  For this discovery work this simple k8s deployment gives me a simple and easily recreatable starting point.

[feedback?](mailto:feedback@hyposcaler.io)
