 # Telepresence sample

 This illustrate how to use [telepresence](https://telepresence.io/) to expose services that are running in a local kuberentes environment to your laptop. 
 Normally you have to mess around with things like port-forwarding etc, but this is unnecessary with telepresence. 
 Once installed it can automatically expose your services via your laptops DNS. 

 Kind allows you to run kubernetes services locally. This can be very useful for local debugging. Typically, however this still requires you to go through a docker build/deploy cycle which can take time. Tools like [Skaffold](https://skaffold.dev/) can help with this. They help you have a continuous deploy loop running and will automatically rebuild and redeploy resources to your local cluster for testing. This can provide an great solution for local development in Kubernetes. 

 Telepresnce can also help - both as an easy way to expose services running in your cluster to local browser- or curl-based access. It can also be used to intercept requests 
 to services and workloads and route them to code running directly on your laptop (for example, launched from your IDE). 

 # Pre-requisites

1. Kubectl 
2. Kind
3. Telepresence  - `brew install telepresenceio/telepresence/telepresence-oss`
4. go installed - `brew install go`

## Setup

Create a Kind cluster

```bash
kind create cluster --name=telepresence
```

Install telepresence in the Kind cluster

``` bash
telepresence helm install
```

This will install the traffic manager which can intercept calls. 

## Deploy sample 

```bash
kubectl apply -f resources/deployment.yaml
```

This will deploy a `hello-world` deployment and a service called `service-hello-world` that can route to it. 

Telepresensce does not require Kubernetes service's, but where they exist they take precedence over routing directly to the deployment. 

## Connect

```bash
telepresence connect --namespace default
telepresence list
```

You should see output like the following (your port may vary). 

```text
Connected to context kind-tele, namespace default (https://127.0.0.1:53113)
deployment hello-world: ready to engage (traffic-agent already installed)
```

## Test local access

```bash
curl http://service-hello-world
```

This will print out a message, e.g:

```text
Hello world!
```

As you can see you can now curl these services directly from your laptop (or via your laptop's browser). 
The message returned is from nginx deployment (defined in the associated configmap)

This mechanism is more convenient than having continually setup port-forwarding using `kubectl` when you want to act as
the client and send requests to a service from your commandline or browser.


## Intercept requests on port 80

You can also have requests that are being sent to services redirected and sent instead to a program running on 
your laptop. This is called `intercepting` and it useful if you are developing server applications. To deploy something
in a Kubernetes cluster you must have a container to run it in (e.g. Docker container). So a  development cycle can 
consist of:

1. Make code changes in your IDE
2. Build container image from code. 
3. Push container image to a repository.
4. Update your Kubernetes resources (e.g. deployments to referene the new image) in your cluster (either via Kustomize or Helm deployment). 
5. Wait for your service to be updated
6. Test your service.


An alternative is to:

1. Make code changes in your IDE
2. Run your code locally on your laptop (launched from your IDE), listening on a specific port. 
3. Use telepresence to intercept requests to a service in the Kubernetes cluster and instead direct the request to this port. 

This can for some developers yield a faster dev cycle leading to improved productivity.  

The following code will add an 
intercept to the hello-world deployment and send it to port 8080 on your laptop. Later we will start a small go program that will listen 
on port 8080.  

```
telepresence intercept hello-world --env-file ./hw.env --port 8080 --mount=false
```

This will output some information about the intercepted service, e.g.: 

```text
Using Deployment hello-world
   Intercept name: hello-world
   State         : ACTIVE
   Workload kind : Deployment
   Intercepting  : 10.244.0.8 -> 127.0.0.1
       80 -> 8080 TCP
```

This tells telepresence to start intercepting requests sent to hello-world and redirect the requests to port 8080 running on your local laptop. 
If there were multiple ports on the service or deployment you would need to add some additional information, e.g.  `--port 8080:80` to disambiguate which 
one to intercept (in this case we want port 80). 

The argument `-env-file ./hw.env` tells telepresence to write the pod's envrionment vars to a local file which can be useful if your local process needs to know this information. You can also gain access ot the mounted filesystem of the pod (but in this case we have chosen not to via `--mount=false` argument).

## Run a local process that listens on port 80

In this case we run a simple Go http server 

```bash
 go run hello_world.go &
 ```

 We run this in the background. 

 ## Test the calls have been intercepted 

 ```bash
curl http://service-hello-world
```

Now when we run the curl command we can see that instead of talking to the nginx container the request was intercepted and directed to the local Golang process

```text
Hello from Go! !
```

This illiustrates that the HTTP request from `curl` has been re-directed to the locally launched Golang program listening on port 8080. 


## Clean-up

```bash
telepresence leave hello-world
telepresence quit
kind delete cluster --name=telepresence
```


# More information


Note: Telepresence can be used with local kind clusters and remote Kubernetes clusters. 

1. https://telepresence.io/docs/quick-start
2. https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/
3. https://support.d2iq.com/hc/en-us/articles/5359773168916-A-simple-Kubernetes-learning-environment-with-KinD-and-Telepresence