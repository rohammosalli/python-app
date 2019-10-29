#### Part 1 
###### Setup Kubernetes Cluster 

we can use GKE and Teraform to Deploy our Cluster or just we can use Kubespray

if you want use Teraform to deploy GEK 

mkdir creds
cp DOWNLOADEDSERVICEKEY.json creds/serviceaccount.json


vim provider.tf

```.yaml
provider "google" {
  credentials = "${file("./creds/serviceaccount.json")}"
  project     = "amir-251909"
  region      = "europe-north1"
  zone        = "europe-north1-a"

}
```
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
terraform plan

terraform apply

I used this link [High Available Kubernetes Cluster Setup using Kubespray](https://schoolofdevops.github.io/ultimate-kubernetes-bootcamp/cluster_setup_kubespray/) to deploy my cluster on Scaleway 


### Part 2 

I used python and Flask framework to develop app and Flask Promethues exporter to expost my python metrics 

if you want to run this app localy please use this method 

```bash 

pip install -r /app/requirements.txt

python app.py 

```
the path /app1 and my /app2 will be exposed in Ingress file 

### Part 3 
 
for each app, I write a Helm Chart, in my chart I tried to use a comment to make everything clear but after run all charts if you find any error or warning something like this 

```bash
error: unable to recognize "deployment": no matches for kind "Deployment" in version "extensions/v1beta1"
```

It's because of Kubernetes version since I used Kubernetes 1.16 some API versions are deprecated you can follow 

[this link ](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)

### part 4 

###### deploy Ingress Controller and some security 

The Best Practice is we pull all nodes behind the firewall

```bash
helm install stable/nginx-ingress --namespace kube-system --name nginx  --set controller.hostNetwork=true,controller.kind=DaemonSet, --set controller.service.externalTrafficPolicy=Local
```

if we want to Using nginx-ingress controller to restrict access by IP we need do some thing, The default value of controller.service.externalTrafficPolicy in the nginx ingress helm chart is ‘Cluster’, we need to change this value to ‘Local’. With the default value of ‘Cluster’ the ingress controller does not see the actual source ip from the client request but an internal IP. After setting this value to ‘Local’ the ingress controller gets the unmodified source ip of the client request.

the we can set our rule in ingress 
```yaml
annotations:
    ingress.kubernetes.io/whitelist-source-range: 49.36.X.X/32

```
To deploy Ingress path securly we can use whitelist or Basic-auth, you can change whitelist in Values file


if you want use Basic-Auth you can add this line to the Jenkinsfile to make it easy 

```bash
sh "htpasswd -b -c password username password" 
sh "kubectl create secret generic basic-auth --from-file=password"
sh "kubectl -n b2c create secret generic basic-auth  --from-file=password --dry-run=true -o yaml | kubectl apply -f -"
```
Then you can use Ingress Anotation 

```yaml
annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropiate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```
But myself I just used whitelist-source-range and alsow nodes behind a firewall    

### part 5  
###### Deploy Prometheus 

To Install prometheus we can use helm and prometheus Document

we need change something in values for example add our domain in Ingress part 

```yaml
 ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - prom.pinsvc.net
```
since we need just Prometheus we can disable other part line alert manager and also if we don't have any storage we can disable PCV in values.yaml

```bash 

helm install --name promethus -f values.yaml .

```

if we want the Prometheus scarp data form our app with /metrics we can set some anotations in our application service, I added this in srvice helm cnthart this will help promethus service discovery to find our endpoint 

```yaml
 name: {{ template "python-app1.fullname" . }}
  annotations:
  # this 4 line will enable metrics fotr our service so promethus can scrap data
    prometheus.io/path: /metrics
    prometheus.io/port: "8080"
    prometheus.io/scheme: http
    prometheus.io/scrape: "true"
```



### part 6 
###### Setup our Jenkins CI CD 

What I did I used a vm for our Jenkins and custome Docker Image 



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



USER jenkins
```
```bash
docker build -t jenkins .
docker tag jenkins rohammosalli/jenkins:lts
docker push rohammosalli/jenkins:lts
```
What I did ? 

I Build my custom Image because I need Helm and Kubectl command and copying my Kubernetes certificate to the Jenkins VM, for security we need setup SSH Key-based authentication and whitelist the specific IP's need to access to this machine.

I just used Jenkins Master, It's not recommended but in Production, we need some Jenkins Slave to do to our jobs


```bash
docker run   -u root   --rm   -d   -p 8080:8080   -p 50000:50000   -v jenkins-data2:/var/jenkins_home   -v /var/run/docker.sock:/var/run/docker.sock -v /root/.kube:/root/.kube   rohammosalli/jenkins:lts
```

this part of docker run -v ```/root/.kube:/root/.kube``` will mount our Kubernetes certificate from Jenkins host to the Jeknins container  

Then I wrote my Jenkinsfile to Run Pipeline it's very simple because I didn't have any advanced experience with Jenkins, Bu I know we can use Built-in variable and shared library to make our Jekninsfile reusable for other Project and also 