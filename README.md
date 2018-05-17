# Drone Kubernetes 
[![Build Status](https://drone.ptrk.io/api/badges/Sh4d1/drone-kubernetes/status.svg)](https://drone.ptrk.io/Sh4d1/drone-kubernetes) [![](https://images.microbadger.com/badges/image/sh4d1/drone-kubernetes.svg)](https://hub.docker.com/r/sh4d1/drone-kubernetes/ "Get your own image badge on microbadger.com")

Drone plugin to create/update Kubernetes resources.

It uses the latest k8s go api, so it is intened to use on Kubernetes 1.9+. I can't guarantee it will work for previous versions.

You can directly pull the image from [sh4d1/drone-kubernetes](https://hub.docker.com/r/sh4d1/drone-kubernetes/)
## Supported resources
Currently, this plugin supports:
* apps/v1
  * DaemonSet
  * Deployment
  * ReplicaSet
  * StatefulSet
* apps/v1beta1
  * Deployment
  * StatefulSet
* apps/v1beta2
  * DaemonSet
  * Deployment
  * ReplicaSet
  * StatefulSet
* v1
  * ConfigMap 
  * PersistentVolume 
  * PersistentVolumeClaim 
  * Pod 
  * ReplicationController 
  * Service 
* extensions/v1beta1
  * DaemonSet
  * Deployment
  * Ingress
  * ReplicaSet

## Inspiration 

It is inspired by [vallard](https://github.com/vallard) and his plugin [drone-kube](https://github.com/vallard/drone-kube).


## Usage

Here is how you can use this plugin:
```
pipeline:
    deploy:
        image: sh4d1/drone-kubernetes
        kubernetes_template: deployment.yml
        secrets: [kubernetes_server, kubernetes_cert, kubernetes_token]
```

## Secrets

You need to define these secrets before.
```
$ drone secret add --image=sh4d1/drone-kubernetes -repository <your-repo> -name KUBERNETES_SERVER -value <your API server>
```
```
$ drone secret add --image=sh4d1/drone-kubernetes -repository <your repo> -name KUBERNETES_CERT <your base64 encoded cert>
```
```
$ drone secret add --image=sh4d1/drone-kubernetes -repository <your repo> -name KUBERNETES_TOKEN <your token>
```

### How to get values of `KUBERNETES_CERT` and `KUBERNETES_TOKEN`

```
$ kubectl get secret -n <namespace of secret> <name of your drone secret> -o yaml | egrep 'ca.crt:|token:'
```

You can copy/paste the encoded certificate to the `KUBERNETES_CERT` value.
For the `KUBERNETES_TOKEN`, you need to decode it:
* `echo "<encoded token> | base64 --decode"`
* `kubectl describe secret -n <your namespace> <drone secret name> | grep 'token:'`



## Required secrets

```bash
    drone secret add --image=honestbee/drone-kubernetes \
        your-user/your-repo KUBERNETES_SERVER https://mykubernetesapiserver

    drone secret add --image=honestbee/drone-kubernetes \
        your-user/your-repo KUBERNETES_CERT <base64 encoded CA.crt>

    drone secret add --image=honestbee/drone-kubernetes \
        your-user/your-repo KUBERNETES_TOKEN eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJ...
```

When using TLS Verification, ensure Server Certificate used by kubernetes API server
is signed for SERVER url ( could be a reason for failures if using aliases of kubernetes cluster )

## How to get token
1. After deployment inspect you pod for name of (k8s) secret with **token** and **ca.crt**
```bash
kubectl describe po/[ your pod name ] | grep SecretName | grep token
```
(When you use **default service account**)

2. Get data from you (k8s) secret
```bash
kubectl get secret [ your default secret name ] -o yaml | egrep 'ca.crt:|token:'
```
3. Copy-paste contents of ca.crt into your drone's **KUBERNETES_CERT** secret
4. Decode base64 encoded token
```bash
echo [ your k8s base64 encoded token ] | base64 -d && echo''
```
5. Copy-paste decoded token into your drone's **KUBERNETES_TOKEN** secret

### RBAC

When using a version of kubernetes with RBAC (role-based access control)
enabled, you will not be able to use the default service account, since it does
not have access to update deployments.  Instead, you will need to create a
custom service account with the appropriate permissions (`Role` and `RoleBinding`, or `ClusterRole` and `ClusterRoleBinding` if you need access across namespaces using the same service account).

As an example (for the `web` namespace):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drone-deploy
  namespace: default

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: drone-deploy
rules:
  - apiGroups: ["extensions"]
    resources: ["deployments"]
    verbs: ["get","list","patch","update"]

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: drone-deploy-bind
  namespace: default
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: drone-deploy
  apiGroup: rbac.authorization.k8s.io
```

Once the service account is created, you can extract the `ca.cert` and `token`
parameters as mentioned for the default service account above:

```
kubectl -n web get secrets
# Substitute XXXXX below with the correct one from the above command
kubectl -n web get secret/drone-deploy-token-XXXXX -o yaml | egrep 'ca.crt:|token:'
```

