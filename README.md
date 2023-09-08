# Using Canary splitter with Consul and Argo Rollouts Progressive Delivery

## Requirements
* Kubernetes cluster 
* Helm or Consul K8s CLI (If you don't have Consul previously installed in K8s)

> NOTE: We are using `LoadBalancer` service type for the API Gatewau, so bear in mind having your `LoadBalancer` services available (if you are using a local environment with Minikube you can use it by installing [](MetalLB) or using `minikube tunnel`)
## Installing a test environment

We are including in the repo a `consul.yaml` values file in case that you want to deploy your own local test environment like Minikube, MicroK8s or similar

* You can use `consul-k8s` or `helm`. If you use `consul-k8s`:
  ```
  kubectl create ns consul
  kubectl create secret generic consul-bootstrap-token --from-literal token=ConsulR0cks
  consul-k8s install -f consul/consul.yaml
  ```
* Install Argo Rollouts. [https://argo-rollouts.readthedocs.io/en/stable/installation/#controller-installation](Here are the steps)

> NOTE: This `consul.yaml` is using an Enterprise version of Consul. So you would need to create a `consul-ent-license` secret with the license, or change the images for using Consul Community (previously OSS):
> ```
> ...
> image: hashicorp/consul:1.16.1
> ...
> ```

## Deploying the app and config

This example is deploying the following components to simulate a traffic management for Canary releases:
* An Argo rollout to manage the progressive delivery for canary process
* A [Consul resolver](https://developer.hashicorp.com/consul/docs/connect/config-entries/service-resolver) to differentiate service instances based on Consul service tags
* A [Consul splitter](https://developer.hashicorp.com/consul/docs/connect/config-entries/service-splitter) that splits traffic between `stable` subset and `canary` subset based on a weight configuration
* Consul API Gateway to ingress traffic to the mesh and route to the `frontend` example service


You can use the included `kustomized.yaml` to install all required objects (from the repo directory):

```
kubectl apply -k ./
```

Check that you can see a `frontend` service in Consul with three instances (two of them should be tagged as `stable`)

Now you should be able to access the service:
```
curl http://$(kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[].ip}'):9090
```


You should get the response `Hello from frontend World`. Let's see that we allways receive the same response. (Use Ctrl-C to exit from the following command)

Let's deploy a Canary with Argo Rollouts [kubectl plugin](https://argo-rollouts.readthedocs.io/en/stable/features/kubectl-plugin/)

```
kubectl argo rollouts set image frontend-rollout frontend=hcdcanadillas/pydemo-front:v1.5
```

Check that the rollout is executed from Argo:
```
$ kubectl argo rollouts get rollout frontend-rollout -w

Name:            frontend-rollout
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/3
  SetWeight:     50
  ActualWeight:  50
Images:          hcdcanadillas/pydemo-front:v1.4 (stable)
                 hcdcanadillas/pydemo-front:v1.5 (canary)
Replicas:
  Desired:       2
  Current:       2
  Updated:       1
  Ready:         2
  Available:     2

NAME                                          KIND        STATUS     AGE    INFO
⟳ frontend-rollout                            Rollout     ॥ Paused   27m    
├──# revision:2                                                             
│  └──⧉ frontend-rollout-86bd684cdf           ReplicaSet  ✔ Healthy  5m31s  canary
│     └──□ frontend-rollout-86bd684cdf-42pd2  Pod         ✔ Running  5m31s  ready:2/2
└──# revision:1                                                             
   └──⧉ frontend-rollout-59db886cbb           ReplicaSet  ✔ Healthy  27m    stable
      └──□ frontend-rollout-59db886cbb-q5qs2  Pod         ✔ Running  27m    ready:2/2
```

Once you see the `Canary` rollout heathy you can do a `Ctrl-C`. And you can see in the Consul UI (Services > frontend > Instances) that you have an instance tagged `stable` and another one  tagged `canary`

So now you can access the API Gateway and you can check that 75% of the traffic goes to `stable` and the other 25% to `canary`:

```
curl http://$(kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[].ip}'):9090;
sleep 1;
echo "";
done
```

The output should be something like:
```
Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World<br><br>These are the secret values:<br>No secrets mounted...
Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World

Hello from frontend World
...
```

You can change the `weights` in the file `consul/splitter.yaml` and apply it to see the difference.
