# External-istiod




## Install Istio with an External Control Plane

1. Create 2 clusters, external cluster will host gateway and external-istiod and remote cluster will host workloads.

```sh
export CTX_EXTERNAL_CLUSTER=<your external cluster context>
export CTX_REMOTE_CLUSTER=<your remote cluster context>
export CTX_REMOTE_CLUSTER=<remote cluster name>
```


2. Point kube-config file to api server of external clusters and create gateway in external cluster.This will install ingress gateway and istiod pod in istio-system pod. this istiod pilot pod will configure ingress-gateway.

```sh
kubectl config use-context $CTX_EXTERNAL_CLUSTER
istioctl install -f controlplane-gateway.yaml

```

3. Expose istio ingress gateway of external cluster, This will be used by remote cluster to access istiod.

```sh
➜  external-istiod git:(main) kubectl get svc  istio-ingressgateway -n istio-system

NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                                           AGE
istio-ingressgateway   LoadBalancer   10.36.12.60   34.136.193.103   15021:32584/TCP,15012:32618/TCP,15017:32015/TCP   3h48m

export EXTERNAL_ISTIOD_ADDR=<hostname for istio-ingressgateway LB IP>
```

4. Configure remote cluster

```sh
kubectl config use-context $CTX_REMOTE_CLUSTER
kubectl create namespace external-istiod
#Configure remote-config-cluster.yaml with the EXTERNAL_ISTIOD_ADDR variable.
istioctl manifest generate -f remote-config-cluster.yaml | kubectl apply -f -
kubectl get mutatingwebhookconfiguration
```

5. Install external istio control-plane on external cluster.

```sh
kubectl config use-context $CTX_EXTERNAL_CLUSTER
kubectl create namespace external-istiod
istioctl x create-remote-secret \
  --context="${CTX_REMOTE_CLUSTER}" \
  --type=config \
  --namespace=external-istiod \
  --service-account=istiod \
  --create-service-account=false | \
kubectl apply -f - 
# Configure external-istiod.yaml with EXTERNAL_ISTIOD_ADDR variable.
istioctl manifest generate -f external-istiod.yaml | kubectl apply -f -
kubectl get po -n external-istiod 
kubectl apply -f external-istiod-gw.yam
```



4. Deploy httpbin and sleep application

```sh
kubectl config use-context $CTX_REMOTE_CLUSTER
kubectl create  namespace sample
kubectl label  namespace sample istio-injection=enabled
kubectl apply -f samples/helloworld/helloworld.yaml -l service=helloworld -n sample 
kubectl apply -f samples/helloworld/helloworld.yaml -l version=v1 -n sample 
kubectl apply -f samples/sleep/sleep.yaml -n sample 
kubectl get pod -n sample 
```
