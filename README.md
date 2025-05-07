# How to collect logs with Fluentd


This repository is here to guide you through the GitHub tutorial that goes hand-in-hand with a video available on YouTube and a detailed blog post on my website. 
Together, these resources are designed to give you a complete understanding of the topic.


Here are the links to the related assets:
- YouTube Video: [How to collect logs with Fluentd](https://www.youtube.com/watch?v=j76ozzIbuO8)
- Blog Post: [How to collect logs with Fluentd](https://isitobservable.io/observability/kubernetes/how-to-collect-logs-with-fluentd)


Feel free to explore the materials, star the repository, and follow along at your own pace.

## Tutorial begins

What you'll learn
* How to build a customized Fluentd container with the Dynatrace plugin
* How to deploy Fluentd in a Kubernetes cluster using a Configmap
* How to ingest metrics using the Dynatrace output plugin
* How to chain Fluent Bit and Fluentd

This repository showcases the usage of Fluentd by using GKE with:
* the HipsterShop
* Prometheus
* Istio
* Fluentd
* Dynatrace


## Prerequisite 
The following tools need to be installed on your machine:
- jq
- kubectl
- git
- gcloud (if you're using GKE)
- Helm

## Requirements
If you don't have any Dynatrace tenant, then let's start a [trial on Dynatrace](https://www.dynatrace.com/trial/)

## 1. Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
## 2. Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
## 3. Clone the GitHub repo
```
git clone https://github.com/isItObservable/Episode-10---FluentD-tutorial.git
cd Episode-10---FluentD-tutorial
```
## 4. Deploy the sample Application

### Istio
0. Create the various namespaces
For the hipsterShop :
```
   kubectl create namespace hipster-shop
   kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
```

1. Download Istioctl
```
curl -L https://istio.io/downloadIstio | sh -
```
This command downloads the latest version of Istio (in our case, Istio 1.10.2) compatible with our operating system.
2. Add istioctl to your PATH
```
cd istio-1.10.3
```
This directory contains samples with addons. We will refer to it later.
```
export PATH=$PWD/bin:$PATH
```
### 1. Install Istio

#### a. Deployment of Istio
To enable Istio, you need to install istio with the following settings
 ```
istioctl install --set profile=demo -y
 ```

#### b. Label the hipster-shop namespace

Then we want to instruct Istio to automatically inject the Envoy Proxy into all the pods of our Hipster-shop application
So we will label the namespace: hipster-shop
```
kubectl label namespace hipster-shop istio-injection=enabled
```

### 2.HipsterShop
```
cd hipstershop
./setup.sh
```

#### Update the ingress gateway to expose ports for the sockshop
```
kubectl edit svc istio-ingressgateway -n istio-system
```
Add the following ports :
```
- name: web
  nodePort: 31770
  port: 8080
  protocol: TCP
  targetPort: 8182
```

####  Expose the HipsterShop out of the cluster
```
kubectl apply -f istio/hipstershop_gateway.yaml
```

### 3. Deploy Prometheus
#### 1. Prometheus
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack 
```
#### 2. Expose Grafana
```
kubectl edit svc istio-ingressgateway -n istio-system
```
Add the following ports :
```
- name: grafana
  nodePort: 31775
  port: 8888
  protocol: TCP
  targetPort: 8888
```
#### 3. Expose the Prometheus server
```
kubectl edit svc istio-ingressgateway -n istio-system
```
Add the following ports :
```
- name: prometheus
  nodePort: 31776
  port: 9090
  protocol: TCP
  targetPort: 9090
```

Deploy the gateway and Virtual Services :
```
kubectl apply -f istio/Prometheus_Grafana_gateway.yaml
```
### 4. FluentD

#### 1. Generate the Docker file 

In order to deliver our tutorial, we need a Fluentd version having the following plugins preinstalled :
- the input plugin: forward ( to connect later on fluentbit with fluentd)
- the output plugin: [dynatrace](https://github.com/dynatrace-oss/fluent-plugin-dynatrace)

To combine both plugins, we're going to build the new image based on `fluentd-kubernetes-daemonset:v1.14.1-debian-forward-1.0`

To build the image you need to install Docker on your laptop: [docker desktop](https://www.docker.com/get-started)
```
cd /fluentd
docker build . -t fluentd-dyantrace:0.1
```
The Dockerfile only adds the installation of the library with this command:
```
RUN gem install fluent-plugin-dynatrace -v 0.1.5
RUN gem install fluent-plugin-kubernetes_metadata_filter -v 2.7.2
RUN gem install fluent-plugin-multi-format-parser
RUN gem install fluent-plugin-concat
```
In our tutorial, I have already built the Docker image and pushed it to Docker Hub.
We will use the following image: ```hrexed/fluentd-dyantrace:0.2```
#### 2. Generate a Platform as a Service Token in Dynatrace
The log ingest API of Dynatrace is reachable only from the Active Gate.
To deploy the Active Gate, it would be required to generate a PaaS Token:
In Dynatrace, click:
* Settings
* Integration
* Generate 
* Give a name and copy the value of the PaaS Token
<p align="center"><img src="/image/paas.png" width="60%" alt="dt api scope" /></p>

#### 3. Generate API Token in Dynatrace
Follow the instructions described in the [Dynatrace documentation](https://www.dynatrace.com/support/help/shortlink/api-authentication#generate-a-token)
Make sure that the scope log ingest is enabled.
<p align="center"><img src="/image/api_rigth.png" width="60%" alt="dt api scope" /></p>

#### 4. Get the cluster ID of your K8s cluster
```
kubectl get namespace kube-system -o jsonpath='{.metadata.uid}'
```
#### 5. Update the deployment of Fluentd and of the active gate

* Create a service account and cluster role for accessing the Kubernetes API.
```
kubectl apply -f fluentd/service_account.yaml
```
Create a secret holding the environment URL and login credentials for this registry, making sure to replace.
```
export ENVIRONMENT_URL=<with your environment URL (without 'http'). Example: environment.live.dynatrace.com>
export CLUSTERID=<YOUR CLUSTER ID>
export PAAS_TOKEN=<YOUR PAAS TOKEN>
export API_TOKEN=<YOUR API TOKEN>
export ENVIRONMENT_ID=<YOUR environementid in your environment url>
kubectl create secret docker-registry tenant-docker-registry --docker-server=${ENVIRONMENT_URL} --docker-username=${ENVIRONMENT_ID} --docker-password=${PAAS_TOKEN} -n dynatrace
kubectl create secret docker-registry tenant-docker-registry --docker-server=${ENVIRONMENT_URL} --docker-username=${ENVIRONMENT_ID} --docker-password=${PAAS_TOKEN} -n dynatrace
kubectl create secret generic tokens --from-literal="log-ingest=${API_TOKEN}" -n dynatrace
 ```

Update the file named fluentd/fluentd-manifest.yaml and activegate.yaml, by running the following command :
 ```
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," fluentd/fluentd-manifest.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluentd/fluentd-manifest.yaml
sed -i "s,ENVIRONMENT_URL_TO_REPLACE,$ENVIRONMENT_URL," fluentd/activegate.yaml
 ```

#### 6. Deploy
```
kubectl apply -f fluentd/activegate.yaml
kubectl apply -f fluentd/fluentd-manifest.yaml
```

#### 7. Connect the active Gate to your dynatrace tenant
To get native Kubernetes metrics, you need to connect the Kubernetes API to Dynatrace.

Get the Kubernetes API URL.
```

kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```
Get the bearer token from the dynatrace-monitoring service account.
```
kubectl get secret $(kubectl get sa dynatrace-monitoring -o jsonpath='{.secrets[0].name}' -n dynatrace) -o jsonpath='{.data.token}' -n dynatrace | base64 --decode
```
In the Dynatrace menu, go to Settings > Cloud and virtualization > Kubernetes, and select Connect new cluster.
Provide a Name, Kubernetes API URL, and the Bearer token for the Kubernetes cluster. Note: For Rancher distributions, you need the bearer token that was created in Rancher web UI, as described in Special instructions for Rancher distributions above. Once you connect your Kubernetes clusters to Dynatrace, you can get native Kubernetes metrics, like request limits, and differences in pods requested vs. running pods.

### 4. Logs ingested in dynatrace

The current deployment of Fluentd is collecting the logs from the Kubernetes cluster using the input plugin tail :
```
    <source>
      @id in_tail_container_logs
      @type tail
      tag raw.kubernetes.*
      path /var/log/containers/*.log
      pos_file /var/log/fluentd.pos
      read_from_head true
      
      <parse>
        @type multi_format
        <pattern>
          format json
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        
        <pattern>
          format regexp
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
          expression /^(?<time>.+)\b(?<stream>stdout|stderr)\b(?<log>.*)$/
        </pattern>
        
      </parse>
    </source>
```
Let's have a look a the log ingested in Dynatrace.
Open Dynatrace and click Logs on the left menu .
<p align="center"><img src="/image/logsviewer.PNG" width="60%" alt="dt api scope" /></p>

### 6. Let's add Fluentbit
#### 1. Deploy Fluentbit
```
kubectl create namespace logging
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml
```
#### 2. Update the Fluentd deployment
```
kubectl delete -f fluentd/fluentd-manifest.yaml
```
Now let's use the Fluentd deployment using the input forward plugin
But we need to update all the information to connect to Dynatrace.
Let's update the deployment :
 ```
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," fluentbit/fluentd-manifest_with_fluentbit.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluentbit/fluentd-manifest_with_fluentbit.yaml
 ```
Now we can deploy the new Fluentd log stream pipeline
 ```
kubectl apply -f fluentbit/fluentd-manifest_with_fluentbit.yaml
 ```
#### 3. Deploy Fluent Bit
```
kubectl apply -f fluentbit/fluentbit_deployment.yaml 
```
