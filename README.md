![IndiaNIC-Symbol-Black.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FTemplate&files=logo-inic-black-img.jpg)

# **In-house kubernetes demo apps deployment**



### **Development Server:** 

- hostname: k0s-dev
- server ip: 10.2.1.113 
- username: indianic 
- password: indianic



### **Kubernetes Version:** 

Check the versions by executing **kubectl version** command

- client version: v1.22.2
- Server version: v1.21.3+k0s



## **<u>Demo Apps Deployment:</u>**

For this demo, I have used following apps, load balancer, and ingress.

### **Apps (using YAML configuration):**

- mongodb (mongo database)
- mongo-express (Web UI to access mongo database)
- nginx

### **Load balancer (using YAML configuration):**

- MetalLB

### **Ingress (using YAML configuration):**

- Traefik


![ In-house_kubernetes.jpg](http://git.indianic.com/SKM/D2018-5450/devops-assets/raw/master/Kubernetes%20(in-house)/images/In-house_kubernetes.jpg)






<u>**Note: use following order of yamls to deploy Apps, Load Balancer (MetalLB) and Ingress controller (Traefik)**</u>



Following deployments, secrets, configmaps, and services were used.

**mongodb and mongo-express:** 

1. ​	mongo-secret.yaml (mongodb **secret** for username/password)
2. ​	mongo.yaml (mongodb **deployment** and **service**)
3. ​	mongo-configmap.yaml (**configmap:** mongodb url to access via mongo-express)
4. ​	mongo-express.yaml (mongo-express **deployment** and **service**)



**nginx:**

1. ​	nginx-depl.yaml (nginx **deployment** and **service**)



**MetalLB loadbalancer:**

MetalLB is an external load balancer for kubernetes. To install MetalLB follow the given steps.

1. ​	metallb-namespace.yaml (create **namespace** for MetalLB)
2. ​	metallb.yaml (**PodSecurityPolicy, ClusterRole, ServiceAccount, etc**)
3. ​	metallb-configmap.yaml (external **ip range** and **protocol**)

         ​	protocol: layer2
         ​	ip range: 10.2.45.1-10.2.45.3



**Traefik Ingress Controller:**

Traefik is used with load balancer for the internal routing (ingress controller). To install Traefik follow the given steps.

1. ​	traefik-rbac.yaml (for **ClusterRoleBinding** and **ClusterRole**)
2. ​	traefik-deployment.yaml (**ServiceAccount**, **Deployment** and **Service**)

​           service type of Traefik service should be “loadbalancer” in order to use MetalLB external ip.  



**Ingress Resource:**

To create and apply host-based rule or domain-based rule, one needs to create an ingress resource.

1. ​	 ingress-rules.yml (ingress for the **routing rules**)



**Add hosts file:**

add following entry of hosts line to your hosts file located at /etc/hosts

`10.2.45.1       test.example.com nginx.example.com`

- 10.2.45.1 (Load balancer ip)
- test.example.com (mongo-express)
- nginx.example.com (nginx)
