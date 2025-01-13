# k3s-wordpress-home-lab
This repo will be used to house my documentation and all of the configurations and steps taken to get my home cluster up and running.

The tools I plan to use are as follows:
- ArgoCD for cluster management
- Lens for monitoring and managing the cluster
- Cert-manager for certificate management within the cluster
- Helm for making Kubernetes manifests

## Step 1 - Getting my Raspberry Pi(s) online 
This part is less related to the actual Kubernetes knowledge and more about getting your Pi off the ground so you can start utilizing it. To do this, I set up a Raspberry Pi 4, which is basically plug-and-play, and gave it a static IP through a reservation on my router

Thats it - it's that simple!

## Step 2 - Getting k3s installed on the Pi
To get k3s installed and turn this Pi into a master node, all I had to do was run this command from a shell on the Pi (I chose to ssh in using my daily driver PC) :
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable=traefik --flannel-backend=host-gw --tls-san=192.168.1.85 --bind-address=192.168.1.85 --advertise-address=192.168.1.85 --node-ip=192.168.1.85 --cluster-init" sh -s -
```
From there I had access to kubectl and was technically ready to start building from manifests.
![image](https://github.com/user-attachments/assets/a2cf24c3-c87a-4497-b80a-21d42ee2fe14)

## Step 3 - Getting ArgoCD installed and exposed on an external IP for management from my normal PC
ArgoCD was installed onto the cluster using the following 2 commands:
```
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
This creates a few different pods and services that will be the backbone of ArgoCD's functionality
```
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          7h18m
pod/argocd-applicationset-controller-64f6bd6456-gwk7w   1/1     Running   0          7h18m
pod/argocd-dex-server-5fdcd9df8b-clwbq                  1/1     Running   0          7h18m
pod/argocd-notifications-controller-778495d96f-ccx5q    1/1     Running   0          7h18m
pod/argocd-redis-69fd8bd669-85rsn                       1/1     Running   0          7h18m
pod/argocd-repo-server-75567c944-xspm2                  1/1     Running   0          7h18m
pod/argocd-server-5c768cdd96-55rwr                      1/1     Running   0          7h18m

NAME                                              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP      10.43.54.154    <none>          7000/TCP,8080/TCP            7h18m
service/argocd-dex-server                         ClusterIP      10.43.3.81      <none>          5556/TCP,5557/TCP,5558/TCP   7h18m
service/argocd-metrics                            ClusterIP      10.43.244.199   <none>          8082/TCP                     7h18m
service/argocd-notifications-controller-metrics   ClusterIP      10.43.29.204    <none>          9001/TCP                     7h18m
service/argocd-redis                              ClusterIP      10.43.122.131   <none>          6379/TCP                     7h18m
service/argocd-repo-server                        ClusterIP      10.43.73.115    <none>          8081/TCP,8084/TCP            7h18m
service/argocd-server                             LoadBalancer   10.43.8.167     som.eip.add.res 80:32595/TCP,443:32454/TCP   7h18m
service/argocd-server-metrics                     ClusterIP      10.43.112.21    <none>          8083/TCP                     7h18m
```
As you can see in the output above, I also patched the service/argocd-server to have an external IP so that I can manage it without needing a desktop environment on the Pi using this command:
```
kubectl patch service argocd-server -n argocd --patch '{ "spec": { "type": "LoadBalancer", "loadBalancerIP": "192.168.68.64" } }'
```
Now I can access ArgoCD from my normal PC just by going to the external IP at port 80 in a browser - with no need for a desktop on the Pi or a display hooked up to it.

When you get to the ArgoCD WebUI you will be met with a login prompt. The default username will be admin and you can find your password by running the following command 
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
## Step 5 - Getting Lens set up to monitor and modify my cluster
As far as installing Lens, that can be done directly from their website and is very painless, just like getting a cluster set up - as you will see in a moment.

To get Lens connected to your cluster, all you have to do is cat your kubeconfig which can easily be done with this command:
```
kubectl config view --minify --raw
```
If you are on a Windows machine, you can simply paste the output into a .txt file -- which will look something like this:
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: redacted
    server: https://som.eip.add.res
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: redacted
    client-key-data: redacted
```
From there, you can open Lens, click Kubernetes Clusters > Local Kubeconfigs and click the plus button to the right of Local Kubeconfigs then find your txt file and you will be monitoring your cluster in real time with the ability to manipulate the resources within it. 
