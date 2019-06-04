## [Dynamically create and use a persistent volume with Azure Files in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv)

## step 1
run `azure-file-sc.yaml` to create *kubernetes storage class*

## step 2
run `azure-pvc-roles.yaml` to create *cluster role and binding*

## step 3
run `azure-file-pvc.yaml` to create a *persistent volumne claim*

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

