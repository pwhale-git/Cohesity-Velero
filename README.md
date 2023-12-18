### 1. Prepare Cohesity for K8s backup by cli into Cohesity
Update Cohesity gflags in Cohesity cli to allow for Kubernetes backups:

```
iris_cli cluster update-gflag service-name=magneto gflag-name=magneto_kubernetes_enable_protection_for_k8s_v122 gflag-value=true effective-now=true reason=enableK
```
```
iris_cli cluster update-gflag service-name=magneto gflag-name=magneto_kubernetes_maximum_supported_version gflag-value=1.26 effective-now=true reason=enableK
```
```
iris_cli cluster update-gflag service-name=magneto gflag-name=magneto_kubernetes_velero_version gflag-value=-1 effective-now=true reason=enable
```
```
iris_cli cluster update-gflag gflag-name=magneto_kubernetes_use _internal_ip gflag-value=true service-name=magneto  effective-now=true reason=01451792
```

Make sure that Cohesity cluster has valid SSL Certificate. Refer to documentation for reference;user need Cohesity account to access
[https://docs.cohesity.com/6_6/Web/UserGuide/Content/CLI/UpdateSSLCLI.htm](https://docs.cohesity.com/6_6/Web/UserGuide/Content/CLI/UpdateSSLCLI.htm) 
or contact Cohesity support for help if needed.

***
### 2. Prepare K8s Cluster for Cohesity integration by kubectl 
Create Cohesity service account and assign cluster-admin role to account
```
kubectl create serviceaccount cohesity -n default
```
```
kubectl create clusterrolebinding cohesity-admin --clusterrole=cluster-admin --serviceaccount=default:cohesity
```

Create secret for Cohesity service account in reachable namespace e.g. file name cohesity-secret.yaml in namespace default
```
apiVersion: v1
kind: Secret
metadata:
  name: cohesity
  annotations:
    kubernetes.io/service-account.name: cohesity
type: kubernetes.io/service-account-token
```

Push Datamover, Velero, and Velero plugin for AWS image into private registry.

Use nerdctl to pull image from private registry to ALL K8s workers. [https://github.com/containerd/nerdctl](https://github.com/containerd/nerdctl)

***
### 3. Input parameters from 2. into Cohesity GUI 
Format for input on GUI is `https://<IP of K8s master node>:6443`

Insert Bearer token from cohesity using `kubectl describe secret cohesity`

Insert image component from registry:
* datamover image (datamover version == Cohesity cluster version)
* velero image
* aws-plugin image

*** 
### 4. Integrate with harderned K8s cluster
Modify Velero deployment:
```
kubectl edit deployment velero -n <namespace of Cohesity cluster>
```

Add the following:
```
~
    spec:
      containers:
      - args:
~
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsGroup: 999
          runAsNonRoot: true
          runAsUser: 1001
~
      initContainers:
~
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsGroup: 999
          runAsNonRoot: true
          runAsUser: 1001
```

***
### 5. Modify /var/ path to be non-exec in ALL K8s workers
Refer to blog post for details on solution: [https://github.com/vmware-tanzu/velero/discussions/3248](https://github.com/vmware-tanzu/velero/discussions/3248)

```
cat /etc/fstab
mount -o remount,exec /var
edit fstab to -> exec
```

Example:
![image](https://github.com/pwhale-git/Cohesity-Velero/assets/150215442/b0b0db2e-0b52-4400-a08c-4db1ff14cf69)


### Troubleshooting

1. Datamover service not found 
![image](https://github.com/pwhale-git/Cohesity-Velero/assets/150215442/16742c38-4870-4f6a-ba7f-5df24c84404f)

Soln patch Datamover to not run on control plane and to be able to be deployed across K8s workers:
```
kubectl patch daemonset cohesity-dm -n <cohesity-namespace> --patch '{"spec": {"template": {"spec": {"tolerations": [{"effect": "NoSchedule","key": "node-role.kubernetes.io/controlplane","operator": "Exists"},{"effect": "NoExecute","key": "node-role.kubernetes.io/etcd","value": "true","operator": "Equal"}]}}}}'
```

