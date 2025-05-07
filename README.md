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

```
kind create cluster --name=telepresence
```

Install telepresence in the Kind cluster

``` 
telepresence helm install
```

This will install the traffic manager which can intercept calls. 

## Deploy sample 

```
kubectl apply -f resources/deployment.yaml
```

This will deploy a `hello-world` deployment and a service called `service-hello-world` that can route to it. 

Telepresensce does not require Kubernetes service's, but where they exist they take precedence over routing directly to the deployment. 

## Connect

```
telepresence connect --namespace default
telepresence list
```

You should see output like the following (your port may vary). 

```
Connected to context kind-tele, namespace default (https://127.0.0.1:53113)
deployment hello-world: ready to engage (traffic-agent already installed)
```

## Test local access

```
> curl http://service-hello-world
Hello world!
```

As you can see you can now curl these services directly from your laptop (or via your laptop's browser). 
The message returned is from nginx (defined in the configmap)

## Choose to intercept requests on port 80

```
> telepresence intercept hello-world --env-file ./hw.env --port 8080 --mount=false
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

```
 go run hello_world.go &
 ```

 We run this in the background. 

 ## Test the calls have been intercepted 

 ```
 > curl http://service-hello-world
Hello from Go! !
```

Now when we run the curl command we can see that instead of talking to the nginx container the request was intercepted and directed to the local go process


## Clean-up

```
telepresence leave hello-world
telepresence quit
kind delete cluster --name=telepresence
```


# More information


Note: Telepresence can be used with local kind clusters and remote Kubernetes clusters. 

1. https://telepresence.io/docs/quick-start
2. https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/
3. https://support.d2iq.com/hc/en-us/articles/5359773168916-A-simple-Kubernetes-learning-environment-with-KinD-and-Telepresence