# Deploy a simple multi-container application with a DB on a public AKS cluster. 


## This exercise includes:
- Create an Azure Kubernetes Service cluster
- Choose the best deployment options for you Pods
- Expose Pods to internal and external network users
- Configure SSL/TLS for Azure Kubernetes Service ingress
- Monitor the health of an Azure Kubernetes Service cluster
- Scale your application in an Azure Kubernetes Service cluster

## Pre-requisites:
1. You already have an active Azure Subscription
2. You have a Github account and you can perform basic operations
3. You have basic understanding of git operations.

## Lists of tasks:
1.	Use AKS to deploy a Kubernetes cluster.
2.	Configure an Azure Container Registry to store application container images.
	(Deploy the three ratings application components)
3.	Deploy the Fruit Smoothies website document database using Helm 3.
4.	Deploy the Fruit smoothies RESTFul API using deployment manifests.
5.	Deploy the Fruit smoothies website frontend using deployment manifests.
	--------
6.	Deploy Azure Kubernetes ingress using Helm 3.
7.	Configure SSL/TLS on the controller using cert-manager.
8	Configure Azure Monitor for containers to monitor the Fruit Smoothies website deployment.
9.	Configure cluster autoscaler and horizontal pod autoscaler for the Fruit Smoothies cluster.

## Steps:

1. Use AKS to deploy a Kubernetes cluster.

	a. Declare the variables

		REGION_NAME=eastus
		RESOURCE_GROUP=aksworkshop
		SUBNET_NAME=aks-subnet
		VNET_NAME=aks-vnet

	b. Create a new RG
	 
		az group create \
			--name $RESOURCE_GROUP \
			--location $REGION_NAME

	c. Create a vnet

		az network vnet create \
			--resource-group $RESOURCE_GROUP \
			--location $REGION_NAME \
			--name $VNET_NAME \
			--address-prefixes 10.0.0.0/8 \
			--subnet-name $SUBNET_NAME \
			--subnet-prefix 10.240.0.0/16

	d. Store the subnet ID

		SUBNET_ID=$(az network vnet subnet show \
			--resource-group $RESOURCE_GROUP \
			--vnet-name $VNET_NAME \
			--name $SUBNET_NAME \
			--query id -o tsv)

	e. Get the latest aks version

		VERSION=$(az aks get-versions \
			--location $REGION_NAME \
			--query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
			--output tsv)

	f. Get a random aks cluster name

		AKS_CLUSTER_NAME=aksworkshop-$RANDOM
		echo $AKS_CLUSTER_NAME
		

	g. Create a new aks cluster with with latest kubernetes

		az aks create \
		--resource-group $RESOURCE_GROUP \
		--name $AKS_CLUSTER_NAME \
		--vm-set-type VirtualMachineScaleSets \
		--load-balancer-sku standard \
		--location $REGION_NAME \
		--kubernetes-version $VERSION \
		--network-plugin azure \
		--vnet-subnet-id $SUBNET_ID \
		--service-cidr 10.2.0.0/24 \
		--dns-service-ip 10.2.0.10 \
		--docker-bridge-address 172.17.0.1/16 \
		--generate-ssh-keys

	h. retrieve the creds

		az aks get-credentials \
			--resource-group $RESOURCE_GROUP \
			--name $AKS_CLUSTER_NAME

	i. Get nodes, get service, name spaces and create a new one
		
		kubectl.get nodes
		kubectl get svc
		kubectl get namespace
		kubectl create namespace ratingsapp

2. Configure an Azure Container Registry to store application container images.

	a. Create a container registry

		ACR_NAME=acr$RANDOM
		az acr create \
			--resource-group $RESOURCE_GROUP \
			--location $REGION_NAME \
			--name $ACR_NAME \
			--sku Standard

	b. Build the container images using ACR tasks - build the ratings-api image

		git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-api.git
		cd mslearn-aks-workshop-ratings-api
		
	c. build the image and push the image to the ACR

		az acr build \
			--resource-group $RESOURCE_GROUP \
			--registry $ACR_NAME \
			--image ratings-api:v1 .
			
	>**Note**: the name name of the acr registry $ACR_NAME.azurecr.io/ratings-api:v1

	d. build the ratings-web image
	
		cd ~
		git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-web.git
		cd mslearn-aks-workshop-ratings-web
		az acr build \
			--resource-group $RESOURCE_GROUP \
			--registry $ACR_NAME \
			--image ratings-web:v1 .
	>**Note**: the acr registry $ACR_NAME.azurecr.io/ratings-web:v1

	e. verify the images

		az acr repository list \
			--name $ACR_NAME \
			--output table

	f. Configure the aks cluster to auth to the container registry

		az aks update \
			--name $AKS_CLUSTER_NAME \
			--resource-group $RESOURCE_GROUP \
			--attach-acr $ACR_NAME

3. Deploy the Fruit Smoothies website document database using Helm 3. 

	a. Deploy mongodb.Configure the Helm client to use the stable repository
		
		helm repo add bitnami https://charts.bitnami.com/bitnami
		helm search repo bitnami

		helm install ratings bitnami/mongodb \
			--namespace ratingsapp \
			--set auth.username=<your username>,auth.password=<your pwd>,auth.database=ratingsdb

	>**Note** the output.
		
		
	b. To connect to your database from outside the cluster execute the following commands:

		kubectl port-forward --namespace ratingsapp svc/ratings-mongodb 27017:27017 &
		mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD

	In case you need to uninstall the helm release

			helm uninstall ratings --namespace ratingsapp

	c. Create a kubernetes secret to hold the mongodb details

		kubectl create secret generic mongosecret \
			--namespace ratingsapp \
			--from-literal=MONGOCONNECTION="mongodb://<your username>:<your pwd>@ratings-mongodb.ratingsapp:27017/ratingsdb"

4. Deploy the Fruit smoothies RESTFul API using deployment manifests. 
	a. Create a kubernetes deployment for the ratings api.
		
		code ratings-api-deployment.yaml

	>Next, Paste the context from the ratings-api-deployment.txt file
	In this file, update the <acrname> value in the image key with the name of your Azure Container Registry instance.
	Apply the configuration by using the kubectl apply command

		kubectl apply \
			--namespace ratingsapp \
			-f ratings-api-deployment.yaml

	You can watch the pods rolling out using the -w flag

		kubectl get pods \
			--namespace ratingsapp \
			-l app=ratings-api -w

	>If the pods aren't starting, aren't ready, or are crashing, you can view their logs by using *kubectl logs <pod name> --namespace ratingsapp* and *kubectl describe pod <pod name> --namespace ratingsapp*.
		
	b. Check the status of the deployment.

		kubectl get deployment ratings-api --namespace ratingsapp
		kubectl get endpoints ratings-api --namespace ratingsapp

	c. Create a Kubernetes service for the ratings API service - Our next step is to simplify the network configuration for the application workloads. we'll use a Kubernetes service to group your pods and provide network connectivity.

		code ratings-api-service.yaml
		
	>copy paste the content from the ratings-api-service.txt and apply the configuration.

		kubectl apply \
			--namespace ratingsapp \
			-f ratings-api-service.yaml

	d. Check the status of the service

		kubectl get service ratings-api --namespace ratingsapp
		kubectl get endpoints ratings-api --namespace ratingsapp

5. Deploy the Fruit smoothies website frontend using deployment manifests.

	a. Create a Kubernetes deployment for the ratings web front end. Create a file called ratings-web-deployment.yaml by using the integrated editor.

		code ratings-web-deployment.yaml

	>copy paste the content from the ratings-web-deployment.txt and apply the configuration
		
		kubectl apply \
		--namespace ratingsapp \
		-f ratings-web-deployment.yaml

		kubectl get pods --namespace ratingsapp -l app=ratings-web -w
		kubectl get deployment ratings-web --namespace ratingsapp

	b. Create a file called ratings-web-service.yaml by using the integrated editor.

		code ratings-web-service.yaml
		
	>Copy paste the content from the ratings-web-service.txt and apply the configuration
		
		kubectl apply \
		--namespace ratingsapp \
		-f ratings-web-service.yaml

	Check the status
		
		kubectl get service ratings-web --namespace ratingsapp -w

	c. Browse the application using the frontend IP and explore

-----------------------

6.	Deploy Azure Kubernetes ingress using Helm 3 and reconfigure the ratings web service to use ClusterIP (internal ip).

	a. Create a namespace for the ingress.

		kubectl create namespace ingress
		
	b. Configure the Helm client to use the stable repository

		helm repo add stable https://kubernetes-charts.storage.googleapis.com/
		
	c. Install nginx-ingress using HELM repo you already downloaded earlier for MongoDB.

		helm install nginx-ingress stable/nginx-ingress \
		--namespace ingress \
		--set controller.replicaCount=2 \
		--set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
		--set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
		
	d. check the nginx-ingress service created with the public Ip

		kubectl get service nginx-ingress-controller --namespace ingress -w
		
	e. Edit the file called ratings-web-service.yaml and change the **type: to ClusterIP** from **type: LoadBalancer**
		
		code ratings-web-service.yaml

	f. Delete and recreate the ratings-web service
		
		kubectl delete service \
		--namespace ratingsapp \
		ratings-web
		
		kubectl apply \
		--namespace ratingsapp \
		-f ratings-web-service.yaml
		
	g. Create an ingress resource to apply configuration of the routes. To do this, create a file named ratings-web-ingress.yaml and copy the content of the ratings-web-ingress.yaml.txt file. Also, update the puplic ip of the ingress controller in the file.
		
		code ratings-web-ingress.yaml
		

	h. Apply the configuration and deploy the ingress route in the ratingsapp namespace.

		kubectl apply \
		--namespace ratingsapp \
		-f ratings-web-ingress.yaml
		
	i. Open the public ip of the ingress to test the application 

7.	Configure SSL/TLS on the controller using cert-manager.
	Deploy cert-manager by using Helm
	Deploy a ClusterIssuer resource for Let's Encrypt
	Enable SSL/TLS for the ratings web service on Ingress
	Test the application

	a. Deploy cert-manager by using Helm. First create a namespace.

		kubectl create namespace cert-manager
		
	b. Use the Jetstack Helm repository to find and install cert-manager. First, you'll add the Jetstack Helm repository by running the code below.

		helm repo add jetstack https://charts.jetstack.io
		helm repo update
		
	c. Run the following command to install cert-manager by deploying the cert-manager CRD.

		kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.14/deploy/manifests/00-crds.yaml
		
	d. Install the cert-manager Helm chart

		helm install cert-manager \
		--namespace cert-manager \
		--version v0.14.0 \
		jetstack/cert-manager
		
	e. Verify the installation by checking the cert-manager namespace for running pods.

		kubectl get pods --namespace cert-manager
		
	f. Deploy a ClusterIssuer resource for Let's Encrypt. 
		
		code cluster-issuer.yaml
		
	g. Update the file with the content from the cluster-issuer.yaml.txt file and update your email in the *email:* section of the file. 
		
	h. Apply the configuration by using the kubectl apply command. Deploy the cluster-issuer configuration in the ratingsapp namespace.

		kubectl apply \
		--namespace ratingsapp \
		-f cluster-issuer.yaml

	i. Enable SSL/TLS for the ratings web service on Ingress by editing and redeploying the ratings-web-ingress.yaml file.

			code ratings-web-ingress.yaml
		
	>Replace the content of the file with the contents from the file ratings-web-ingress.yaml - withSSL.txt. 
		
	>In this file, update the <ingress ip> value in the host key with the dashed public IP of the ingress you retrieved earlier, for example, frontend.13-68-177-68.nip.io. This value allows you to access the ingress via a host name instead of an IP address.
		
			kubectl apply \
			--namespace ratingsapp \
			-f ratings-web-ingress.yaml
		
	j. Verify that the certificate was issued.

		kubectl describe cert ratings-web-cert --namespace ratingsapp
	
8.	Configure Azure Monitor for containers to monitor the Fruit Smoothies website deployment.
	
	a. Create a Log Analytics workspace

		WORKSPACE=aksworkshop-workspace-$RANDOM
		az resource create --resource-type Microsoft.OperationalInsights/workspaces \
				--name $WORKSPACE \
				--resource-group $RESOURCE_GROUP \
				--location $REGION_NAME \
				--properties '{}' -o table

	b. Enable the AKS monitoring add-on

		WORKSPACE_ID=$(az resource show --resource-type Microsoft.OperationalInsights/workspaces \
		--resource-group $RESOURCE_GROUP \
		--name $WORKSPACE \
		--query "id" -o tsv)
		
		az aks enable-addons \
		--resource-group $RESOURCE_GROUP \
		--name $AKS_CLUSTER_NAME \
		--addons monitoring \
		--workspace-resource-id $WORKSPACE_ID
		
	c. Configure Kubernetes RBAC to enable live log data. You need to create cluster role binder. Create a file named "logreader-rbac.yaml".
		
		code logreader-rbac.yaml
		
	>Copy the content from logreader-rbac.yaml.txt file. Apply it.
		
		kubectl apply \
		-f logreader-rbac.yaml
		
		
	d. Inspect the AKS event logs and monitor cluster health
	View the live container logs and AKS events


9. Configure cluster autoscaler and horizontal pod autoscaler for the Fruit Smoothies cluster.

	a. Create a file called ratings-api-hpa.yaml by using the integrated editor, copy the content from ratings-api-hpa.yaml.txt and apply it

		code ratings-api-hpa.yaml
		
		kubectl apply \
		--namespace ratingsapp \
		-f ratings-api-hpa.yaml
		
	>Important:
		For the horizontal pod autoscaler to work, you must remove any explicit replica count from your ratings-api deployment. Keep in mind that you need to redeploy your deployment when you make any changes.
		To test your autoscaling working, you can follow the steps from:
		Run a load test with [horizontal pod autoscaler enabled](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/10-scale-application).
		
	b. Enable Cluster auto scaler by creating a file called ratings-api-deployment.yaml

		code ratings-api-deployment.yaml
		
		Copy the content from ratings-api-deployment.yaml.txt.
		
		kubectl apply \
		--namespace ratingsapp \
		-f ratings-api-deployment.yaml
		
		kubectl get pods \
		--namespace ratingsapp \
		-l app=ratings-api -w
		
		az aks update \
		--resource-group $RESOURCE_GROUP \
		--name $AKS_CLUSTER_NAME  \
		--enable-cluster-autoscaler \
		--min-count 3 \
		--max-count 5
		
		kubectl get nodes -w
	
10. [More resources](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/11-summary).
	
>Note: This is based on Microsoft [AKS workshop content](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/01-introduction) and intention is to rapidly deploy this setup and create a ready recorner for learning & discussion purposes. There will be added scripts for single click deployments.