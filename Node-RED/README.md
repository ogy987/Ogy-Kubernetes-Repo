# How to deploy Node-RED to Kubernetes cluster on AWS using Helm charts

## STEP 1 – Prerequisites

1) A running Kubernetes cluster with at least 3 nodes (1 master + 2 worker nodes). If you don’t already have one running, use eksctl to get an EKS cluster up and running or use kops to get your Kubernetes cluster.

2) At least **8GB of free memory** and **2 vCPU** in the Kubernetes cluster for Node-RED microservices. An **`m5.large`** instance should do the job.

3) A latest version of kubectl installed, configured, and working on your machine to manage Kubernetes cluster.

	- **Kubectl** is the command-line tool that enables you to execute commands against your Kubernetes cluster. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs.

	- If you are on macOS and using Homebrew package manager, you can install kubectl with Homebrew.

	- Before you run the installation command, you need to update Homebrew package manager with:
	
			$ brew update
	
	- Run the installation command:

			$ brew install kubectl
			
		or
	
			$ brew install kubernetes-cli

	- Test to ensure the version you installed is up-to-date:

			$ kubectl version --client

4) Helm v3 installed

	- **Helm** is a tool that streamlines installing and managing Kubernetes applications. Think of it like Apt/Yum/Homebrew for K8S. Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

	- If you are on macOS and using Homebrew package manager, you can install kubectl with Homebrew.

	- Before you run the installation command, you need to update Homebrew package manager with:

			$ brew update

	- Run the installation command:

			$ brew install helm

	- Test to ensure the version you installed is up-to-date:

			$ helm version


## STEP 2 – Create a Route53 domain for Node-RED

- I used **Certificate Manager** in AWS to create a new record (DNS domain) inside `getswift-testing.co` hosted zone on AWS Dev account.


## STEP 3 –  Install NGINX Ingress Controller

- In this step we are going to install **NGINX Ingress controller** for load balancing and SSL/TLS termination using Helm charts from this URL: https://bitnami.com/stack/nginx-ingress-controller/helm

- Get bitnami's repo for helm:

```bash
	$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

- Install NGINX Ingress controller using helm with `ingress-controller-values-production.yaml` file:

```bash
	$ helm install ingress bitnami/nginx-ingress-controller -f ingress-controller-values-production.yaml
```

- Now, we need to test NGINX, in our web browser, by entering Ingress Controllers EXTERNAL-IP, in order to see if installation was successful. You can find NGINX Ingress Controllers EXTERNAL-IP by running the following command:

```bash
	$ kubectl get services
```

- You will see EXTERNAL-IP for LoadBalancer that NGINX Ingress controller deployment created (something like this `xxxxxxxxxxxxxxxxxxxxxxxxxxxxx.eu-west-1.elb.amazonaws.com`).


## STEP 4 –  Setup Ingress for Node-RED

- After we install NGINX Ingress controller, we need to create **Record Set** inside Route53 `getswift-testing.co` hosted zone for our previously created Node-RED DNS domain and configure the ingress routing.

- In Route53, choose `getswift-testing.co`, find Node-RED record, select it and then goes to **Create Record → Routing policy: Simple routing → Define simple record → Record name: → Value/Route traffic to: Alias to Application and Classic Load Balancer → Choose region: → Choose load balancer: →  Record type: A → Define simple record**.


## STEP 5 –  Install cert-manager

- **cert-manager** is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources and one of them is Let’s Encrypt. It will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry. 

- cert-manager runs within your Kubernetes cluster as a series of deployment resources. It utilizes CustomResourceDefinitions (**CRD**s) to configure Certificate Authorities and request certificates. 

- The CRDs allows you to define and install custom resources into the Kubernetes API. A custom resource is an extension of the Kubernetes API and it represents a customization of a particular Kubernetes installation.

- We are going to install and deploy cert-manager, using Helm charts, by following these steps:

	- Create the namespace for cert-manager:

			$ kubectl create namespace cert-manager

	- Add the Jetstack cert-manager Helm repository:

			$ helm repo add jetstack https://charts.jetstack.io

	- Update your local Helm chart repository cache:

			$ helm repo update

	- Install CRDs resources manually, using kubectl:
	
			$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml
	
	- Install cert-manager using this command:
	
			$ helm install cert-manager --namespace cert-manager jetstack/cert-manager --version v0.14.1
		
> **NOTE** cert-manager requires a number of CRD resources to be installed into your cluster as part of installation. This can either be done manually, using kubectl, or using the installCRDs option when installing the Helm chart.

- Once you’ve installed cert-manager, you can verify it is deployed correctly by checking the cert-manager namespace for running pods:

```bash
	$ kubectl get pods -n cert-manager
```

- Results:
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

- You should see the `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook` pod in a Running state. It may take a minute or so for the TLS assets required for the webhook to function to be provisioned. This may cause the webhook to take a while longer to start for the first time than other pods.

**⚠️ Warning:⚠️**  You should not install multiple instances of cert-manager on a single cluster. This will lead to undefined behavior and you may be banned from providers, such as Let’s Encrypt.

- Once cert-manager has been deployed, you must configure Issuer or ClusterIssuer resources which represent certificate authorities (CAs) from which signed x509 certificates can be obtained.


## STEP 6 –  Install Lets Encrypt cluster issuer

- **`Issuers`**, and **`ClusterIssuers`**, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All cert-manager certificates require a referenced issuer resource (e.g. **Lets Encrypt**) that is in a ready condition to attempt to honor the request. By using the non-namespaced `ClusterIssuer` resource, cert-manager will issue certificates that can be consumed from multiple namespaces. **Lets Encrypt** uses the ACME protocol to verify that you control a given domain name and to issue you a certificate.

- `ClusterIssuers` will instruct cert-manager to issue certificates using the **Lets Encrypt**. Regarding to that, you will need at least one `Issuer`, or `ClusterIssuer` in order to begin issuing certificates within your cluster.

- Add `ClusterIssuer` for Let’s Encrypt, using `letsencrypt-prod.yaml` file, with this command:

```bash
	$ kubectl apply -f letsencrypt-prod.yaml
```

## STEP 7 – Deploy Node-RED using Helm charts

- We will now install Node-RED using the k8s-at-home Helm chart and customized `values.yaml` file:

```bash
	$ helm repo add k8s-at-home https://k8s-at-home.com/charts/

	$ helm repo update

	$ helm install nodered k8s-at-home/node-red -f nodered-values-SSL.yaml
```

- In order to verify that your Node-RED installation was successfully deployed, run the command:

```bash
	$ kubectl get pods
```

- Output like this confirms the successful installation of Node-RED:

```bash
NAME                                READY   STATUS      RESTARTS   AGE
nodered-node-red-66d97b7d55-klnhb   1/1     Running     0          11m
```

- You can check it by accessing your Node-RED GUI inside the browser.
