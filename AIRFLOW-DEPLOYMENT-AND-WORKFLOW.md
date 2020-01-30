# Overview

*Disclaimer*
By the time I wrote this documentation, I didn't plan to pretend to be an expert of the terms and/or topics listed here. I wrote this along learning it.

Apache Airflow, a platform to programmatically author, scheduler and monitor workflows, has gained its popularity in the world of data engineering and machine learning. Its use cases can be
1. Define and monitor Cron jobs
2. Automate certain DevOps operation
3. Move data periodically
4. Machine learning pipeline

After years contribution from open source community, Apache Airflow has rich features to meet normal needs from different organization. In the meantime, it also is flexible enough to meet custom requirement.

In the post, we are focusing on the deployment part of Apache Airflow.

From my limited research, business units within YOUR-COMPANY has been deployed Apache Airflow differently. For instance, *a.com* relies on HashiCorp's service. *b* deployed with *in house devOps*. As part of our plan, we mean to deploy with *Azure Kubernetes Services(AKS)* and *Azure Container Registries (ACR)* to work better with *Azure Blob Storage* and *Azure Data Bricks*.


In general, there are few requirements for us to deploy an Apache Airflow within YOUR-COMPANY.
1. Scalable
*CeleryExecutor* is one of the ways. And we use *CeleryExecutor*.

2. Authentication 
It will be ideal that we could use *Active Directory* to authenticate a YOUR-COMPANY User. Unfortunately, Airflow doesn't provide an *Active Directory* plugin for us to use directly. *LDAP* is supported but we need to understand how it is setup by our IT. (Will be followed-up.)

3. Reliable
Apache Airflow with *CeleryExecutor* needs to have a fairly complicated deployment. For instance, the following 5 services will be deployed
* airflow-web
* airflow-postgresql
* airflow-redis
* airflow-flower
* airflow-worker

We want to make sure all those services could be deployed automatically one time. No need to manual deploy them one by one. Or the service could recover itself or at least retry to do so. Kubernetes is our choice.

4. TLS Termination
After deployed, we need to be able to access the services within YOUR-COMPANY with a domain and *https* service. *Ingress Controller* of *Kubernetes* can be used for this purpose.

5. Able to sync new DAGs
*DAGs* - a Directed Acyclic Graph – is a collection of all the tasks we want to run, organized in a way that reflects their relationships and dependencies.
In Apache Airflow, *DAGs* can be synced either by a *git-sync* or by mounting a shared persistent volume. For sure, we can build a new Docker Container every time while a new *DAG* created or updated. But it might not be the best way to do so.

A better practice for naming a *DAG* is that **always name a new DAG even though you just try to update something from an existing *DAG* because the Airflow will work with the new DAG even though your old DAG is in the process.

We will use the approach of *mounting a shared persistent volume*.

6. Work with a typical workflow (Dev, test and Production)
A better practice is that we need to set up an environment for development, staging and production, respectively. 
A *DAG* is also needed to go through *pull-request* process to be moved into the *master* branch. To better facility of the local development, a local Kubernetes service are recommended to set up. It includes *minikube* and *virtualbox*. Pleas refer to [this](https://kubernetes.io/docs/setup/minikube/) about this setup.

In the end, we are supposed to generate a *Helm* Kubernetes deployment. You should be able to run it locally, in the *AKS* development cluster. The *Staging* and *Production* should be handled by our *DevOps* pipeline. 

7. Behind *YOUR-COMPANY* firewall
Apache Airflow provides *Reverse Proxy*. (Will be followed up)

# Airflow Configuration and Docker Image
Airflow can be configured via *airflow.cfg* file. You can read more [here](https://airflow.apache.org/howto/set-config.html).
The *Unofficial* Airflow Docker Image is maintained by Matthieu "Puckel_" Roisil at *[puckel/docker-airflow](https://github.com/puckel/docker-airflow)*. All variable in *airflow.cfg* can be configured by *Docker environment variable*. 
If you want to experiment around Airflow configuration, I think *Docker environment variable* is easier way.
For instance, I want to change Airflow Postgres connection string, and put it into a *.env* file, and run *Docker run* in this way:
```
docker run --env-file .env --rm --build-arg AIRFLOW_DEPS="datadog,dask" --build-arg PYTHON_DEPS="flask_oauthlib>=0.9" -t puckel/docker-airflow .
```

the *.evn* would look like
```
AIRFLOW__CORE__SQL_ALCHEMY_CONN=YOUR-SQL-ALCHEMY-CONN-STRING
```

# Installation with Helm/Charts
Installing & deploying with *Helm/Charts* would be the easiest way. The official Airflow Helm Chart could be found [here](https://github.com/helm/charts/tree/master/stable/airflow). 
Actually, if you have *Helm* installed locally, you don't need to download this chart, you can just run 
```
helm install --namespace "airflow" --name "airflow" stable/airflow
```
It should install and deploy the airflow with *pucketl/docker-airflow* docker images to your default Kubernetes cluster. 

However, you might see other issues while running this command. Please refer to *Common Error* for fixing.

Most likely, you need to tweak some Airflow configuration, you can do it by modifying *values.xml*, and deploying with your *values.xml*. The following command helps you do so:

```
helm upgrade -f values.yaml \
	             --install \
	             --debug \
	             YOUR-AIRFLOW-NAME \
	             .
```

If you have made any change, you don't have to delete all existed services. You might need to just delete the service has been changed, and use the following 
```
kubectl delete service SERVICE-NAME-YOU-WANT-TO-DELETE
```
then, run
```
helm apply -f values.yaml \
	             --install \
	             --debug \
	             YOUR-AIRFLOW-NAME \
	             .
```

# MiniKube & AKS Deployment
*MiniKube* can be setup by following this [documentation](https://kubernetes.io/docs/setup/minikube/). Once it has been set up correctly, the default Kubernetes is *minicube*
You can see it by running
```
kubectl config get-contexts
```

For AKS, we assume the `Azure CLI` & `kubectl` has been installed locally. If you have not installed `kubectl`, `Azure CLI` can help you

``` install kubectl

az aks install-cli

```

## Create an Azure AKS Service

1. log in: portal.azure.com

2. search ‘aks’ and follow the on-screen instruction to set up an AKS service. You can also use this [doc.](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) as a reference

3. Get credentials to connect to kubernetes cluster using kubectl

```

az aks get-credentials — name YOUR-AKS-SERVICE-NAME — resource-group YOUR-RESOURCE-GROUP

```

4. Show a dashboard of Kubernetes clusters in a web browser

```

az aks browse — name YOUR-AKS-SERVICE-NAME — resource-group YOUR-RESOURCE-GROUP

```

## Switch between MiniKube and AKS
You can use the following command
```
kubectl config get-contexts
```
You might see 
```
minikube
YOUR-AKS-SERVICE-NAME
```
then, use
```
kubectl config use-context YOUR-AKS-SERVICE-NAME
```
so you can deploy it to your AKS service

## Ingress Controller
# Use Helm to deploy an NGINX ingress controller
```
helm install stable/nginx-ingress \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```
If you search around, you might be run into [Microsoft's ingress controller documentation](https://docs.microsoft.com/en-us/azure/aks/ingress-basic). It will work for most of part. Just need to pay attention that it sets a *namespace*, but when we deploy our *airflow*, we didn't specify a *namespace*, the namespace is *default*.

Use this command to see how the nginx-ingress goes
```
 kubectl get service -l app=nginx-ingress
 NAME                                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
aspiring-labradoodle-nginx-ingress-controller        LoadBalancer   10.0.61.144    40.117.74.8   80:30386/TCP,443:32276/TCP   6m2s
aspiring-labradoodle-nginx-ingress-default-backend   ClusterIP      10.0.192.145   <none>        80/TCP                       6m2s
```
We can use the following *ingress controller* configuration file for our AKS airflow *web* service.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: YOUR-airflow-ingress-NAME
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: $1
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: web
          servicePort: 8080
        path: /admin(/|$)(.*)
      - backend:
          serviceName: flower
          servicePort: 5555
        path: /(.*)
```

And we need to modify *values.xml* a little bit to use the same IP or domain for both *web* and *flower* dashboard.
```values.xml line of 252
ingress:
  ## enable ingress
  enabled: true
  ## Configure the webserver endpoint
  web:
    path: ""
    host: ""
    annotations:
    tls:
      ## Set to "true" to enable TLS termination at the ingress
      enabled: false
  ##
  ## Configure the flower endpoint
  flower:
    path: "/flower(/|$)(.*)"
    livenessPath: /
    ## hostname for flower
    host: ""
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
    tls:
      enabled: false
```


Then move to *40.117.74.8* from your browser to see the *web* service. and *40.117.74.8/flower* to see the *flower* service

## Mount a Shared Persistent Volume
I would recommend using this document: [Dynamically create and use a persistent volume with Azure Files in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv) to understand more. 
In short, you will use `install-azure-file-storage.md` in this repo to get your started.
## step 1
run `azure-file-sc.yaml` to create *kubernetes storage class*

## step 2
run `azure-pvc-roles.yaml` to create *cluster role and binding*

## step 3
run `azure-file-pvc.yaml` to create a *persistent volumn claim*

## step 4
run 
```
kubectl get pvc azurefile
``` 

to view the status of the PVC

````
$ kubectl get pvc azurefile
|------|------------|------------------------------------------|----------|--------------|--------------|-----|
|NAME  |      STATUS|    VOLUME                                   |  CAPACITY |  ACCESS MODES |  STORAGECLASS |  AGE|
|azurefile |  Bound |    pvc-8436e62e-a0d9-11e5-8521-5a8664dc0477 |  5Gi      |  RWX          |  azurefile    |  5m  |
```

Then, open the `values.yaml` file, change line `372` to `storageClass: azurefile`. `azurefile` is what you defined `azure file class` name.

# Common Error

## *Error: could not find tiller*
```
kubectl -n kube-system delete deployment tiller-deploy

kubectl -n kube-system delete service/tiller-deploy

helm init
```

## *Error: found in requirements.yaml, but missing in charts/ directory: postgresql, redis*

```
helm dependency update
```

## See whether Pod is ready
The service is *not* immediate available once we run a command. You can check the either service status or pod status by running  
```
kubectl get pods
```

Or 
```
kubectl get services
```