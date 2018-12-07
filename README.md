# knative101
Lab instructions for knative101 - Knative on IKS


## Create IBM Cloud Account and Get Cluster

## Access the Cluster
Learn how to set the context to work with your cluster by using the `kubectl` CLI, access the Kubernetes dashboard, and gather basic information about your cluster.

1.  Set the context for your cluster in your CLI. Every time you log in to the IBM Cloud Kubernetes Service CLI to work with the cluster, you must run these commands to set the path to the cluster's configuration file as a session variable. The Kubernetes CLI uses this variable to find a local configuration file and certificates that are necessary to connect with the cluster in IBM Cloud.

    a. List the available clusters.

    ```shell
    ibmcloud cs clusters
    ```

    b. Download the configuration file and certificates for your cluster using the `cluster-config` command.

    ```shell
    ibmcloud cs cluster-config <your_cluster_name>
    ```

    c. Copy and paste the output command from the previous step to set the `KUBECONFIG` environment variable and configure your CLI to run `kubectl` commands against your cluster.

    Example:
    ```shell
    export KUBECONFIG=/Users/user-name/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml
    ```

2.  Get basic information about your cluster and its worker nodes. This information can help you manage your cluster and troubleshoot issues.

    a.  View details of your cluster.

    ```shell
    ibmcloud cs cluster-get <your_cluster_name>
    ```

    b.  Verify the worker nodes in the cluster.

    ```shell
    ibmcloud cs workers <your_cluster_name>
    ibmcloud cs worker-get <worker_ID>
    ```

3.  Validate access to your cluster.

    a.  View nodes in the cluster.

    ```shell
    kubectl get node
    ```

    b.  View services, deployments, and pods.

    ```shell
    kubectl get svc,deploy,po --all-namespaces
    ```

## Install Istio, Knative, and Kaniko Build Template

Knative is currently built on top of both Kubernetes and Istio.  You will need to install Istio.

1. Install Istio:

	```
	kubectl apply --filename https://github.com/knative/serving/releases/download/v0.2.2/istio.yaml
	```
2. Label the default namespace with `istio-injection=enabled`:

	```
	kubectl label namespace default istio-injection=enabled
	```

3.  Monitor the Istio components until all of the components show a `STATUS` of
    `Running` or `Completed`:

    ```
    kubectl get pods --namespace istio-system --watch
    ```

After installing Istio, you can install Knative.  For this lab, we will install both the Build & Serving components of Knative.

1. Install Knative:

	```
	kubectl apply --filename https://github.com/knative/serving/releases/download/v0.2.2/release.yaml
	```

2. Monitor the Knative components until all of the components show a `STATUS` of `Running` or `Completed`:

	```
	kubectl get pods --namespace knative-serving
	kubectl get pods --namespace knative-build
   ```

## Clone the lab repo
The application for this lab is a simple node.js with express app which returns the first n numbers of the fibonacci sequence.  To use the app, host it, and simply make a POST request to the `/fib` endpoint with the JSON data: `{"number":10}`

1. Clone the git repository:

	```
	git clone https://github.com/beemarie/fib-knative.git
	```
2. Change directories to the fib-knative folder.

	```
	cd fib-knative
	```

## Provide Container Registry Credentials
This lab will need credentials for authenticating to your container registry - we'll be using dockerhub.  You could also use the IBM Container Registry. First, we'll need to create a `Secret` to store the credentials.

A `Secret` is a Kubernetes object containing sensitive data such as a password, a token, or a key. You can also read more about [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

1. To create this object, we'll first need to base64 encode our username and password for dockerhub.

	```
	echo -n "username" | base64 -b 0

	echo -n "password" | base64 -b 0
	```

2. Update the `docker-secret.yaml` file with your base64 encoded username and password.
3. Apply the secret to your cluster:

	```
	kubectl apply --filename docker-secret.yaml

	```

A `Service Account` provides an identity for processes that run in a Pod. This Service Account will be used to link the build process for Knative to the Secret you created earlier.

1.  Apply the service account to your cluster:

	```
	kubectl apply --filename service-account.yaml
	```

## Update Domain Configurations and Ingress Forwarding
When a Knative application is deployed, Knative will define a URL for your application.  By default, this url is "default.example.com." Because we want our application to be accessible at a URL we own, we need to configure Knative to assign new applications to our own hostname.

What hostname should we use? Luckily for us, IBM Kubernetes Service gave us an external domain when we created our cluster.  We'll first get that URL, tell Knative to assign new applications to that URL, and then forward requests to our Ingress Subdomain to the Knative Istio Gateway.

### Update domain configuration for Knative
1. First, let's get our ingress subdomain for our cluster.

	```
	ibmcloud ks cluster-get my-cluster-name
	```

2. Next, update the default URL for new Knative apps by editing the configuration:

	```
	kubectl edit cm config-domain --namespace knative-serving
	```

3. Change instances of `default.example.com` to your ingress subdomain, which should look like: `bmv-knative.us-east.containers.appdomain.cloud`.

### Forward requests coming into IKS ingress to the Knative Istio Gateway

1. Update the forward-ingress.yaml file with your own ingress subdomain, prepended with fib-knative, or whatever subdomain you would like your application to live at.  The file should look something like:

	```yaml
	  apiVersion: extensions/v1beta1
	  kind: Ingress
	  metadata:
	    name: iks-knative-ingress
	    namespace: istio-system
	    spec:
	      rules:
	        - host: fib-knative.default.bmv-knative.us-east.containers.appdomain.cloud
	          http:
	            paths:
	              - path: /
	                backend:
	                  serviceName: knative-ingressgateway
	                  servicePort: 80
	```

2. Apply the ingress rule.

	```
	kubectl apply --filename forward-ingress.yaml
	```


## Deploy app using Kubectl and service.yaml
Let's get our first Knative application up & running. Using the Build & Serving components of Knative, we can go from some source code on github to a docker image built on cluster (using the Kaniko build template, to a docker image pushed to dockerhub, and ultimately a URL to access our application.

1. Edit the service.yaml file to point to your own container registry.

2. Apply the service.yaml file to you cluster.

	```
	kubectl apply -f service.yaml
	```
3. Run `kubectl get pods --watch` to see the pods initializing.

4. Once all the pods are initialized, go to dockerhub to see that your container image was built and pushed to dockerhub.

5. Now that the app is up, we should be able to call it using a number input.  We can do that using a curl command against the URL provided us:

	```
	curl -X POST \
  http://fib-knative.default.bmv-knative.us-east.containers.appdomain.cloud/fib \
  -H 'Content-Type: application/json' \
  -d '{"number":20}'
	```

6. If we left this alone for some time, it would scale itself back down to 0, and terminate the pods that were created.  The default for knative scale-to-zero is 5 minutes.  Let's decrease this time by editing the autoscaler:

	```
	kubectl edit cm config-autoscaler --namespace knative-serving
	```

7. Find `scale-to-zero-threshold`, and decrease the time from 5m to 1m.  You can also decrease the `scale-to-zero-graceperiod`.

8. Run `kubectl get pods --watch` and wait to see the application scale itself back down to 0.

## Deploy vnext version using Knctl

## Canary Testing with Knctl
