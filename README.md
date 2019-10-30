#### Part 1 
###### Setup Kubernetes Cluster 

we can use Teraform to Deploy our GKE Cluster or just we can use Kubespray to deploy self hosted cluster

if you want use Teraform to deploy GEK follow this steps 


Download your service account from GKE dashboard IAM
```bash
mkdir creds
cp DOWNLOADEDSERVICEKEY.json creds/serviceaccount.json
```
you need to create a provider.tf file 

vim provider.tf

```.yaml
provider "google" {
  credentials = "${file("./creds/serviceaccount.json")}"
  project     = "amir-251909"
  region      = "europe-north1"
  zone        = "europe-north1-a"

}
```
you need to create a gke-cluster.tf file 

vim gke-cluster.tf

```.yaml
resource "google_container_cluster" "gke-cluster" {
  name     = "standard-cluster-1"
  location = "europe-north1-a"
  remove_default_node_pool = true
  initial_node_count = 1
  timeouts {
    create = "30m"
    update = "20m"
  }
}

resource "google_container_node_pool" "primary_preemptible_nodes" {
  name       = "gke-node-pool"
  location   = "europe-north1-a"
  cluster    = "${google_container_cluster.gke-cluster.name}"
  node_count = 3
  timeouts {
    create = "30m"
    update = "20m"
  }
}
```
Then you can deploy cluster on GCP with ```terraform apply``` command
```bash
terraform plan

terraform apply
```
###### Self Hosted Cluster 

I used this link [High Available Kubernetes Cluster Setup using Kubespray](https://schoolofdevops.github.io/ultimate-kubernetes-bootcamp/cluster_setup_kubespray/) to deploy my cluster on Scaleway 

1 - One node Master 
2 - One node Worker 

Best Practice is 3 Master Node and 3 Worker Node at minimum 

### Part 2 - Application 

I used Python and Flask framework to develop our app and Flask-Prometheus exporter to expose my python metrics so we can monitor it with Prometheus


###### Run application localy for test

if you want to run this app localy please use this method 

```bash 
pip install -r /app/requirements.txt
python app.py 
```
you can access to the application with / in local, if you run it on Kubernetes the app will be access with example.com/app1

### Part 3 - Helm Chart
 
for each app, I write a Helm Chart, in my chart I tried to use a comment to make everything clear but after run all charts if you find any error or warning something like this 

```bash
error: unable to recognize "deployment": no matches for kind "Deployment" in version "extensions/v1beta1"
```

It's because of Kubernetes version since I used Kubernetes 1.16 some API versions are deprecated you can follow 

[this link ](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)

#### Note:
1. if you want change something you can try to edit values.yaml
2. if you have compelex path you need add something in Ingress file 


You have to change / to /$1 if you have complex path 
```yaml 
    nginx.ingress.kubernetes.io/rewrite-target: /$1
```    
```yaml
path: /app1/?(.*)
```

You may find something like this in templates/deployment.yaml, I used podAntiAffinity to make our application more efficient and Cluster Balanced

```yaml
 affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - {{ template "python-app1.fullname" . }}
              topologyKey: kubernetes.io/hostname
            weight: 100
```            
### part 4 

###### deploy Ingress Controller and some security 

The Best Practice is we pull all nodes behind the firewall


Exposing applications using services
There are five types of Services:
1. ClusterIP (default)
2. NodePort
3. LoadBalancer
4. ExternalName
5. Headless

We used ClusterIP for our applications and LoadBalancer for our Ingress Controller 

```bash
helm install stable/nginx-ingress --namespace kube-system --name nginx  --set controller.hostNetwork=true,controller.kind=DaemonSet, --set controller.service.externalTrafficPolicy=Local
```
###### Node security

By default, Google Kubernetes Engine nodes use Google's Container-Optimized OS as the operating system on which to run Kubernetes and its components. Container-Optimized OS implements several advanced features for enhancing the security of Google Kubernetes Engine clusters, including:

1. Locked-down firewall
2. Read-only filesystem where possible
3. Limited user accounts and disabled root login
4. Use key-based Authentication

###### Limiting Pod-to-Pod communication
By default, all Pods in a cluster can be reached overnetwork via their Pod IP address, we can use Ingress and egress and network policis to allow use tags to define the traffic flowing through your Pods, Once a network policy is applied in a namespace, all traffic is dropped to and from Pods that don't match the configured labels. 

###### Pod Security Policy
We need to be sure we are using security context on pods and containers, for example all containers should run as none user root, all Pods in a cluster adhere to a minimum baseline policy that you define.


##### Ingress Security 
if we want to Using nginx-ingress controller to restrict access by IP we need do some thing, The default value of controller.service.externalTrafficPolicy in the nginx ingress helm chart is ‘Cluster’, we need to change this value to ‘Local’. With the default value of ‘Cluster’ the ingress controller does not see the actual source ip from the client request but an internal IP. After setting this value to ‘Local’ the ingress controller gets the unmodified source ip of the client request.

Then we can set our rule in ingress 
```yaml
annotations:
    ingress.kubernetes.io/whitelist-source-range: 49.36.X.X/32
```
To deploy Ingress path securly we can use whitelist or Basic-auth, you can change whitelist in Values file


if you want use Basic-Auth you can add this line to the Jenkinsfile to make it easy 

```bash
sh "htpasswd -b -c password username password" 
sh "kubectl -n b2c create secret generic basic-auth  --from-file=password --dry-run=true -o yaml | kubectl apply -f -"
```
But I already added in Jenkinsfile
```yaml
annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropiate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```
###### Use HTTPS 
Now that you have enabled external access to our apps our any instance, the next step is to enable HTTPS for our domains in Kubernetes, we can use Let’s Encrypt Certificates and Cert-Manager, for example, if you have an API and you send your  user and password in POST request from front to backend without any SSL it can be hacked but using SSL will encrypt your HTTP body request 


### part 5  
###### Deploy Prometheus 

To Install prometheus we can use helm

We need change something in values for example add our domain in Ingress part and enable ingress, disable other stuff if you don't need 

```yaml
 ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - prom.pinsvc.net
```
Since we need just Prometheus we can disable others like alert manager and also if we don't have any storage we can disable PCV in values.yaml

```bash 
#clone the heelm chart then run this 
helm install --name prometheus -f values.yaml .

```
If we want the Prometheus scarp data form our app with /metrics we can set some anotations in our application service, I added this in srvice helm cnthart, this will help prometheus service discovery to find our endpoint 

```yaml
 name: {{ template "python-app1.fullname" . }}
  annotations:
  # this 4 line will enable metrics fotr our service so prometheus can scrap data
    prometheus.io/path: /metrics
    prometheus.io/port: "8080"
    prometheus.io/scheme: http
    prometheus.io/scrape: "true"
```
After this for example you can find this ```(python_gc_collections_total)``` query in promethesu PromQL TextBox or you can find it in kubernetes-services at Prometehus Service Discovery 

### Part 6 
###### Setup our Jenkins CI/CD

If you want deploy Jenkins on Kubrnetes you can take look at [google solutions](https://cloud.google.com/solutions/jenkins-on-kubernetes-engine) 

But what I did! I used a VM To Install Docker and Used Custome Jenkins image, we have some ways to Integrate Jenkins with 
Kubernetes with some plugins, But I used manual way because it's was easy since I don't use GKE

```Dockerfile
FROM jenkins/jenkins:lts
USER root
RUN apt-get update && \
    apt-get -y install apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common && \
    curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
    add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
    $(lsb_release -cs) \
    stable" && \
    apt-get update && \
    apt-get -y install docker-ce
RUN usermod -a -G docker jenkins

RUN KUBE_VERSION= curl -S https://storage.googleapis.com/kubernetes-release/release/stable.txt
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -S https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && \
    chmod +x ./kubectl && \
    mv ./kubectl /usr/local/bin/kubectl

RUN curl -LO https://kubernetes-helm.storage.googleapis.com/helm-v2.10.0-linux-amd64.tar.gz && \
    tar -xzf helm-v2.10.0-linux-amd64.tar.gz && \
    mv ./linux-amd64/helm /usr/local/bin/helm && \
    rm -rf helm-v2.10.0-linux-amd64.tar.gz

RUN apt-get install apache2-utils -y 

USER jenkins
```
```bash
docker build -t jenkins .
docker tag jenkins rohammosalli/jenkins:lts
docker push rohammosalli/jenkins:lts
```

I Build my custom Image because I need Helm and Kubectl command and copying my Kubernetes certificate to the Jenkins VM, for security we need setup SSH Key-based authentication and whitelist the specific IP's need to access to this machine.

###### I just used Jenkins Master, It's not recommended but in Production, we need some Jenkins Slave to do to our jobs


```bash
docker run   -u root   --rm   -d   -p 8080:8080   -p 50000:50000   -v jenkins-data2:/var/jenkins_home   -v /var/run/docker.sock:/var/run/docker.sock -v /root/.kube:/root/.kube   rohammosalli/jenkins:lts
```

This part of docker run ```-v /root/.kube:/root/.kube``` will mount our Kubernetes certificate from Jenkins host to the Jeknins container  

Then I wrote my Jenkinsfile to Run Pipeline it's very simple, because I didn't have any advanced experience with Jenkins, But I know we can use``` Built-in variable``` and ```shared library``` to make our Jekninsfile reusable for other Project.

