## DENY all traffic from other namespaces

__(_a.k.a  LIMIT access to the current namespace_)__

You can configure a NetworkPolicy to **deny all the traffic from other
namespaces while allowing all the traffic coming from the same namespace** the
pod deployed to.

**Use Cases**
- You do not want deployments in `test` namespace to accidentally
  send traffic to other services or databases in `prod` namespace.
- You host applications from different customers in separate Kubernetes
  namespaces and you would like to block traffic coming from outside a
  namespace.

![Diagram of DENY all traffic from other namespaces policy](img/4.gif)

### Example

Create a new namespace called `secondary` and start a web service:

```
kubectl create namespace secondary

kubectl run web --namespace secondary --image=nginx \
    --labels=app=web --expose --port 80
```

Save the following manifest to `web-deny-other-namespaces.yaml` and apply
to the cluster:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: secondary
  name: web-deny-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

```
$ kubectl apply web-deny-other-namespaces.yaml
networkpolicy "web-deny-other-namespaces" created"
```

Note a few things about this manifest:

- `namespace: secondary` deploys it to the `secondary` namespace.
- it applies the policy to ALL pods in `secondary` namespace as the
  `spec.podSelector.matchLabels` is empty and therefore selects all pods.
- it allows traffic from ALL pods in the `secondary` namespace, as
   `spec.ingress.from.podSelector` is empty and therefore selects all pods.

## Try it out

Query this web service from the `default` namespace:

```sh
$ kubectl run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.secondary
wget: download timed out
```

It blocks the traffic from `default` namespace!

Any pod in `secondary` namespace should work fine:

```sh
$ kubectl run test-$RANDOM --namespace=secondary --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.secondary
<!DOCTYPE html>
<html>
```

### Cleanup

    kubectl delete deployment web -n secondary
    kubectl delete service web -n secondary
    kubectl delete networkpolicy web-deny-other-namespaces -n secondary
    kubectl delete namespace secondary
