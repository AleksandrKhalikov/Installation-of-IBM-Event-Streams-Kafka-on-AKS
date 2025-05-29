# Installation of IBM Event Streams Kafka on AKS
This is a step-by-step instruction of how to install [IBM Event Streams Kafka](https://www.ibm.com/products/event-automation) on a plain K8S cluster in AKS

## Step 1 - Connect kubeclt to your AKS Cluster

Follow an official instruction - [AKS quick-kubernetes-deploy-cli](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli#:~:text=Connect%20to%20the%20cluster)

```
az aks get-credentials --resource-group $MY_RESOURCE_GROUP_NAME --name $MY_AKS_CLUSTER_NAME
kubectl get nodes
```

## Step 2 - Perform installation in Terminal:

Open your preffered terminal
Perform the folowing:

```
helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm

kubectl create namespace aks-event-streams

helm install eventstreams ibm-helm/ibm-eventstreams-operator -n aks-event-streams
```

Get your IBM Entitelment key from here - https://myibm.ibm.com/products-services/containerlibrary

Continue with the installation:

```
kubectl create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=<YOUR-IBM-ENTITELMENT-KEY> --docker-server="cp.icr.io" -n aks-event-streams

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux \
  --set "controller.extraArgs.enable-ssl-passthrough="
```

## Step 3 - CR.yaml

Get a template from here - https://github.com/IBM/ibm-event-automation/tree/main/event-streams/cr-examples/eventstreams/kubernetes

On AKS unlike EKS DNS names for ingress endpoints are not automatically provisioned unless you are using ingress controller integrated with a DNS provider (like ExternalDNS with Azure or a managed ingress addon). Since AKS doesn't auto generate DNS, you may use nip.io as a wildcard DND svc. This is an accepted solution for dev and non prod env.

https://nip.io/

I will use develompent.yaml (this one - https://github.com/IBM/ibm-event-automation/blob/main/event-streams/cr-examples/eventstreams/kubernetes/development.yaml)
You need to change class: <INGRESS-CLASS> to host: nginx and replace host: <HOSTNAME> to an ip address of your ingress service

```
kubectl get svc -n ingress
```
In my case:

![3d78e321-da4c-48ab-bf5e-52d728351b38](https://github.com/user-attachments/assets/b8c75905-8001-431d-91ab-a2ec4e174828)


Save it to a new development-fixed.yaml file (like the one in this repository) and apply it:
```
kubectl apply -f development-fixed.yaml -n aks-event-streams

kubectl get eventstreams -n aks-event-streams
```

You will see this:

![29d87533-fb9d-48af-a4b7-9a43a0ee10d3](https://github.com/user-attachments/assets/25b20c5e-a19d-465a-9591-0c17463dfe9e)


## Step 4 - Create an Admin User

According to documentation, in the next step you need to [install Event Streams CLI](https://ibm.github.io/event-automation/es/installing/post-installation/#:~:text=to%20become%20ready.-,Installing%20the%20Event%20Streams%20command%2Dline%20interface,-The%20Event%20Streams)

To do that, you need to login to Event Streams as an Administrator, but you don't have Administrator user yet. You need to create it.
To create an admin user, do this, apply es-admin-kafkauser.yaml from this repository and restast event streams operator to load new credentials:

```
kubectl apply -f es-admin-kafkauser.yaml
### TO PICKUP NEW CREDENTIALS
kubectl rollout restart deployment/eventstreams-cluster-operator -n aks-event-streams
kubectl get kafkauser -n aks-event-streams
```

You will see something like this:

![60e2c58a-2f15-4189-b728-5c841fc4a363](https://github.com/user-attachments/assets/11884a54-28af-4516-a44d-51cd97f1004d)


es-admin kafkauser was created.

Now, you need to get the password.
Do the following:

```
kubectl get secret es-admin -n aks-event-streams -o jsonpath="{.data.password}" | base64 -d
```



Remove % char at the end. This is your admin password
Now you can login to Event Stream Admin UI:

https://adminui.eventstreams.YOUR_IP_FROM_PREVIOUS_STEP.nip.io/
username: es-admin
password: from previous step (don't forget to remove % at the end)

## Well Done

![ef2e15c1-deae-4581-aa60-b57fcd6f9619](https://github.com/user-attachments/assets/1e15f859-6354-4da4-b535-01840e8b5b77)

Now you can proceed with [installation on Event Streams CLI](https://ibm.github.io/event-automation/es/installing/post-installation/#:~:text=to%20become%20ready.-,Installing%20the%20Event%20Streams%20command%2Dline%20interface,-The%20Event%20Streams)
