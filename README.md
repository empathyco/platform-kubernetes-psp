# ops-kubernetes-psp
Deploy PSP policies in Kubernetes cluster


# GKE 

For this exercise, two GKE clusters will be needed, one with PSP disabled and other one with PSP enabled.

```sh
gcloud beta container clusters create test-1 

gcloud beta container clusters create test --enable-pod-security-policy --num-nodes=1
```

# Static Pod

Deploy a pod in some nodes of the k8s cluster would be possible using the node directory ```/etc/kubernetes/manifests``` as hostPath in a pod example and downloading there some yamls to deploy resources directly by the kubelet daemon without the API server observing them. 

The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.

Check in GKE with PSP disabled

```sh 
 kubectl apply -f deployments/malicious.yaml
 kubens org-1
 kubectl exec -it test-a /bin/bash 
    cd /malicious
    apt-get update
    apt-get install wget 
    wget https://raw.githubusercontent.com/empathyco/ops-kubernetes-psp/main/deployments/pod-example.yaml
```

Check the containers which are currently running in the node using:

```sh
docker ps | grep POD_pause
```

# Pod Security Policy



![image](https://rancher.com/img/blog/2020/pod-security/picture1.png)

Using PSPs you can: 

* Prevent privileged pods from starting and control privilege escalation.
* Restrict access to the host namespaces, network and filesystem the pod can access.
* Restrict the users/groups a pod can run as.
* Limit the volumes a pod can access.
* Restrict other parameters like runtime profiles or read-only root filesystems.



![image](https://rancher.com/img/blog/2020/pod-security/picture2.png)

In summary:

* Pod Identity determined by its **service account**
  * If **not declared** in the spec, default service account will be used
* Allow the **use** of a PSP declaring a **Role** or **ClusterRole**
* Needs to be a **RoleBinding** that associates the **Role** (which allow the access to use the PSP) with the **service account** declared in the pod/deploy spec
  
## Examples

The examples above will be tested using two kind of PSP:
* Privileged
* Unprivileged

```sh
kubectl apply -f policies/privileged-psp.yaml

kubectl apply -f policies/unprivileged-psp.yaml
```

### Nginx Example

* Running as Pod using privileged service account
  * Check info
* Running as Pod using unprivileged service account
  * Check errors 
* Running as Pod using an unprivileged image with an unprivileged service account 
* Running as Deployment


```sh
kubectl --as=system:serviceaccount:org-1:privileged -n org-1 apply -f deployments/nginx.yaml

kubectl --as=system:serviceaccount:org-1:unprivileged -n org-1 apply -f deployments/nginx.yaml

kubectl apply -f deployments/nginx-deployment.yaml

kubectl delete po test-a --force --grace-period=0
```

### Malicious Example 

* Running as Pod using an unprivileged image with an unprivileged service account and mount the hostPath ```/etc/kubernetes/manifests```
  * Check errors


```sh 
kubectl --as=system:serviceaccount:org-1:unprivileged -n org-1 apply -f deployments/nginx-unprivileged-malicious.yaml
kubectl describe po test-a 
```

## Tips 

* Dockerfile 
  * Use ```USER <UID>``` instead of ```USER <name>``` 
* Use ```kubectl --as=system:serviceaccount:org-1:privileged -n org-1 auth can-i use psp/privileged``` to check access to certain PSP


# References 

* [Create Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
* [TGI Kubernetes 078: Pod Security Policies](https://youtu.be/zErhwjPRKn8)
* [Kubernetes Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
* [Using pod security policies GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/pod-security-policies)
* [Enhancing Kubernetes Security with Pod Security Policies, Part 1](https://rancher.com/blog/2020/pod-security-policies-part-1)
* [Enhancing Kubernetes Security with Pod Security Policies, Part 2](https://rancher.com/blog/2020/pod-security-policies-part-2)
* [PSP Users and groups](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#users-and-groups)
