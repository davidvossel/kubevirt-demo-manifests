# kubevirt-demo-manifests

Contains various install and VM demo manifests

# Install

## KubeVirt dev environment ENV VARs

```
export KUBEVIRT_PROVIDER=k8s-1.21
export KUBEVIRT_NUM_NODES=2
export KUBEVIRT_STORAGE=rook-ceph-default
```

## Install kubevirt + cdi (example using v0.42.1)

```

echo "Installing CDI"
export CDI_RELEASE=v1.35.0
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_RELEASE/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_RELEASE/cdi-cr.yaml


echo "Installing KubeVirt"
export KV_RELEASE=v0.42.1
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${KV_RELEASE}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${KV_RELEASE}/kubevirt-cr.yaml
kubectl -n kubevirt wait kv kubevirt --for condition=Available

echo "Download virtctl client"
wget https://github.com/kubevirt/kubevirt/releases/download/v0.42.1/virtctl-${KV_RELEASE}-linux-amd64
chmod 755 virtctl-${KV_RELEASE}-linux-amd64 
mv virtctl-${KV_RELEASE}-linux-amd64 virtctl

```


# Create a migratable Fedora VM with persistent storage


Create the VM and watch the DV import
```
cat << EOF > vm-alpine-persistent.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-alpine-persistent
  name: vm-alpine-persistent
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-alpine-persistent
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
          interfaces:
          - masquerade: {}
            name: default
        resources:
          requests:
            memory: 1Gi
      terminationGracePeriodSeconds: 0
      networks:
      - name: default
        pod: {}
      volumes:
      - dataVolume:
          name: alpine-dv
        name: datavolumedisk1
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: alpine-dv
    spec:
      pvc:
        accessModes:
        - ReadWriteMany
        volumeMode: Block
        resources:
          requests:
            storage: 2Gi
      source:
        registry:
          url: docker://quay.io/kubevirt/alpine-container-disk-demo:v0.42.1
EOF

kubectl create -f vm-alpine-persistent.yaml

kubectl get dv -w
```

Start the VM and watch it move to Running phase

```
./virtctl start vm-alpine-persistent

kubectl get vmi -w

```

Access the VM's console
```
./virtctl console vm-alpine-persistent
```

# Live Migrate the VM

Enable live migration feature gate

```
cat << EOF > kv-cr.patch
spec:
  configuration:
    developerConfiguration:
      featureGates: ["LiveMigration"]
EOF

kubectl patch kv -n kubevirt kubevirt --type merge --patch="$(cat kv-cr.patch)"

```


Post and watch the migration object
```
cat << EOF > migration.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: vm-alpine-persistent
EOF

kubectl get vmi

kubectl create -f migration.yaml

kubectl get vmi -w
```
