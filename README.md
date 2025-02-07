# k3s-wordpress-home-lab
This repo will be used to house my documentation and all of the configurations and steps taken to get my home cluster up and running.

The tools I plan to use are as follows:
- ArgoCD for cluster management
- Lens for monitoring and managing the cluster
- Cert-manager for certificate management within the cluster
- Helm for making Kubernetes manifests
    - Helm Playground for quickly viewing Helm dry runs
 

# Pre-Project Work and Set-Up



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


# Actual Project Work - Creating Charts, Creating YAMLs, Running Pipelines 


### Step .5 - Creating a secret for the SQL DB

You will need a secret for your database password.

This can be done by simply running this block in your shell:

```
echo "your-password" | base64
```

and copying the output in the shell into a basic YAML that will look roughly like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
type: Opaque
data:
  password: <output-from-shell>
```

Now run a simple ```kubectl apply -f demo-secret.yaml```

You can then pass this secret into your values.yaml so that your SQL database can use it - for example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: <no value>
spec:
  selector:
    matchLabels:
      app: <no value>
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: <no value>
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: MySQL
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: demo-secret
```

## Step 1 - Basic Chart Creation 

First things first, we needed a very basic chart that contained the bare minimum for this Wordpress site to run. As I am new to Helm and self hosting in general, I wanted to see what this thing would need to run at a very basic level and be available on Local Host.

I wanted to write the chart myself instead of taking a bitnami chart and changing values in the values.yaml file because I felt like I would learn more about how charts function by writing it from scratch.

I went and found a very simple chart structure image. This is the one I found the most helpful: 

![image](https://github.com/user-attachments/assets/e8ab9bd4-59d7-4f69-93b3-23295f6c2bf3)

From here, I started creating my YAMLs with very basic deployments, services, volumes, and claims before I even started writing the values.yaml file so that I could see what the k8s side was going to look like before I started integrating Helm. There is also a basic Wordpress Deployment on the k8s docs that is almost exactly identical to what I have created that I referenced: https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

After my YAMLs were complete in a boiler plate state and ready to go, I started working on the values.yaml and I wanted to make sure I was using it to the point of maybe even _overuse_ so that I really got a grasp of how passing in different values works.

For some of the simple values I wrote them into the values file by themselves in just one line like so: 

![image](https://github.com/user-attachments/assets/deeaec71-ce3c-4cc4-be19-08988498c177)

I then referenced those values in the deployment YAMLs like so:

![image](https://github.com/user-attachments/assets/e496bc56-f333-444d-8c19-3c4558fd16e3)

This is extremely handy, intuitive, and intriguing _but_ I knew there had to be something to make this easier for larger sections like spec.cotainer.env where you can have very basic YAMLs that look like this:

```
env:
  mysql:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-pass
        key: password
  - name: MYSQL_DATABASE
    value: wordpress
  - name: MYSQL_USER
    value: wordpress
  - name: MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-pass
        key: password
```

Now, imagine a production environment where there can be substantially more variables.. Manually passing in those values one at a time would take ages.

Fortunately, Helm has an answer for that with the "- toYaml" operator which allows you to pass in entire sections of your values.yaml file to your deployment YAMLs. If I want to pass in the entire codeblock above, I can use something like this:

```
env:
{{- toYaml .Values.env.mysql | nindent 8 }}

```
As long as you have the proper amount of indentation, it will pass the entire section in for you and make your deployment much more readable.


## Step 2 - Getting The Deployments In Order And Confirming Access

Alright, so I have my deployments, services, and volumes in my manifests and I think they are ready to go - now what?

The answer to this was a simple one. Try it.

I failed quite a few times.

First things first, we need to clone this repo on the node itself and then package and install the chart. This is the easy part.

I cloned the repo, did a simple ``` helm package . ``` in the same directory that housed my chart.yaml, values.yaml, and templates directory.

From there you can simply do a ```helm install <desired-name-of-helm-deployment> <packaged-chart>.tgz``` and it will install your chart and deploy your resources.

Once its all deployed, you can run some simple commands like  ```kubectl get pods``` and ```kubectl get services``` to confirm everything is up and running.

Hopefully, you will see "running" on all your pods and all of your expected services will show when you query those.

To confirm access to your wordpress deployment, the first thing to do should be checking your wordpress NodePort service to confirm that you have endpoints.

This can be done with ```kubectl get endpoints wordpress```

If you get an output - great! If there are no inputs, it's time to start debugging. Check and make sure your labels and selectors match up!

In the event that you do get an output, try to curl the endpoint from the node ```curl -I <endpoint-IP>:<NodePort-from-service>``` (you can find your nodeport by doing ```kubectl get service```

Your output should look something like this: ```wordpress         NodePort    10.43.75.98   <none>        8080:**30005**/TCP   74m```

With a NodePort service, your WordPress site should be available on your local network.

To actually visit your website from your local network, go to ```http://<IP-of-node>:<NodePort-from-service>```

## Step 3 - Setting up the Application in ArgoCD


First things first, let's get a manifest created for the application. It'll look something like this:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wordpress    #This will be the name of your ArgoCD application
  namespace: argocd  #This is the namespace that your ArgoCD install can be found in (this shouldnt need to be changed)
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/MaxShaw5/k3s-wordpress-home-lab.git    #This is your GitHub repo
    targetRevision: HEAD #Branch of repo
    path: wordpress #Make sure this leads to the directory where your chart.yaml and values.yaml files are
    helm:
      valueFiles:
        - values.yaml #This should be named whatever your values.yaml files are named - if using multiple you can add them all here
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

Now we want to apply that to the ArgoCD namespace in our cluster

```kubectl apply -n argocd -f <argocd-application>.yaml```

This should build the application within ArgoCD and you can confirm this by looking in the webUI to see if your application is there.

Sync your application using ```argocd app sync wordpress --strategy=apply```

You should now see your application in ArgoCD and it should be sync'd up and ready to start continuously deploying your revisions.

![image](https://github.com/user-attachments/assets/9ea9d03d-537e-4ec6-8bd8-cc4ababcb1bf)


To test the ArgoCD implementation, try changing something in one of your manifests - I chose to use the replicaCount.

First, I did a ```kubectl get pods``` to see that I had 2 pods. Then, I changed the replica count of my wordpress deployment to 1 in my values.yaml file. From there, its a simple merge into main and then ArgoCD automatically updated my cluster and changed it to 1 replica which I confirmed by doing another ```kubectl get pods```


