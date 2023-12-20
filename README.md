# Using Canary splitter with Consul and Argo Rollouts progressive delivery

> [!NOTE]  
> Forked from: https://github.com/dcanadillas/consul-argo-rollouts

## Requirements
* Kubernetes cluster 
* Helm or Consul K8s CLI (if you don't have Consul previously installed in K8s)

> [!NOTE]
> This example uses an API Gateway configured for **Consul 1.16+**. In addition, the example uses the `LoadBalancer` service type for the API Gateway. Iif you are using a local environment with Minikube, use [MetalLB](https://metallb.universe.tf/installation/) or `minikube tunnel` to access the API Gateway.


## Installing a test environment (optional)

This example includes a `consul.yaml` values file so you can deploy your own environment locally (for example, Minikube, MicroK8s, kind).

* Install Consul using `consul-k8s` or `helm`. The following commands installs Consul with `consul-k8s` and sets a bootstrap token.

  ```
  kubectl create ns consul
  kubectl create secret generic consul-bootstrap-token --from-literal token=ConsulR0cks -n consul
  consul-k8s install -f consul/consul.yaml
  ```

> [!NOTE]
> This `consul.yaml` uses Consul Enterprise. You need to create a `consul-ent-license` secret containing the license, or update `consul.yaml` to use Consul Community Edition (previously OSS):
> ```
> ...
> image: hashicorp/consul:1.16.1
> ...
> ```

* Install [Argo Rollouts](https://argo-rollouts.readthedocs.io/en/stable/installation/#controller-installation).

## Deploying the app and config

This example deploys the following components to simulate a traffic management for Canary releases:

* An Argo rollout to manage the progressive delivery for canary process
* A [Consul resolver](https://developer.hashicorp.com/consul/docs/connect/config-entries/service-resolver) to differentiate service instances based on Consul service tags
* A [Consul splitter](https://developer.hashicorp.com/consul/docs/connect/config-entries/service-splitter) that splits traffic between `stable` subset and `canary` subset based on a weight configuration
* Consul API Gateway to ingress traffic to the mesh and route to the `frontend` example service

Install the required components using the `kustomization.yaml` file.

```
kubectl apply -k ./
```

Check that you can see a `frontend` service in Consul with three instances (two of them should be tagged as `stable`)

Now you should be able to access the service.

```
curl http://$(kubectl get svc api-gateway -o jsonpath='{.status.loadBalancer.ingress[].ip}'):9090
```

You should get the response `Hello from frontend World`. 

Now, deploy a Canary with Argo Rollouts [kubectl plugin](https://argo-rollouts.readthedocs.io/en/stable/features/kubectl-plugin/).

```
kubectl argo rollouts set image frontend-rollout frontend=hcdcanadillas/pydemo-front:v1.5
```

Check that the rollout is executed from Argo:

```
kubectl argo rollouts get rollout frontend-rollout -w
```

This command will return something similar to the following:

```
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

Once you see the `Canary` rollout heathy, exit the command by entering `Ctrl-C`. In the Consul UI (Services > frontend > Instances), you should find an instance tagged `stable` and another one tagged `canary`.

Now you can access the API Gateway and you can check that 75% of the traffic goes to `stable` and the other 25% to `canary`.

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
