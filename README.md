# kubevirt-demo-manifests

Folders contain demo scenario manifests

install kubevirt (example using v0.42.2)

```
export RELEASE=v0.42.2
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml
kubectl -n kubevirt wait kv kubevirt --for condition=Available

```

-
