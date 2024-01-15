# This is for test script

Test install rke2 with helm
https://rancher.github.io/rodeo

#############################################################################
Create a Kubernetes cluster for Rancher
Rancher can run on any Kubernetes cluster and distribution, that is certified to be standard compliant by the Cloud Native Computing Foundation (CNCF).

We recommend using a RKE2 Kubernetes cluster. RKE2 is a CNCF certified Kubernetes distribution, which is easy and fast to install and upgrade with a focus on security. You can run it in your datacenter, in the cloud as well as on edge devices. It works great on a single-node as well in large, highly available setups.

In this Rodeo we want to create a single node Kubernetes cluster on the Rancher01 VM in order to install Rancher into it. This can be accomplished with the default RKE2 installation script:

sudo bash -c 'curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_CHANNEL="v1.24" \
  sh -'
 Click to run on Rancher01
Create a configuration for RKE2

sudo mkdir -p /etc/rancher/rke2
sudo bash -c 'echo "write-kubeconfig-mode: \"0644\"" > /etc/rancher/rke2/config.yaml'
 Click to run on Rancher01
After that you can enable and start the RKE2 systemd service:

sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
 Click to run on Rancher01
The service start will block until your cluster is up and running. This should take about 1 minute.

You can access the RKE2 logs with:

sudo journalctl -u rke2-server
 Click to run on Rancher01
Creating a highly available, multi-node Kubernetes cluster for a highly available Rancher installation would not be much more complicated. You can run the same installation script with a couple more options on multiple nodes.

You can find more information on this in the RKE2 documentation.


Testing your cluster
RKE2 now created a new Kubernetes cluster. In order to interact with its API, we can use the Kubernetes CLI kubectl.

To install kubectl run:

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
 Click to run on Rancher01
We also have to ensure that kubectl can connect to our Kubernetes cluster. For this, kubectl uses standard Kubeconfig files which it looks for in a KUBECONFIG environment variable or in a ~/.kube/config file in the user's home directory.

RKE2 writes the Kubeconfig of a cluster to /etc/rancher/rke2/rke2.yaml.

We can copy the /etc/rancher/rke2/rke2.yaml file to our ~/.kube/config file so that kubectl can interact with our cluster:

mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown ec2-user: ~/.kube/config
 Click to run on Rancher01
In order to test that we can properly interact with our cluster, we can execute two commands.

To list all the nodes in the cluster and check their status:

kubectl get nodes
 Click to run on Rancher01
The cluster should have one node, and the status should be "Ready".

To list all the Pods in all Namespaces of the cluster:

kubectl get pods --all-namespaces
 Click to run on Rancher01
All Pods should have the status "Running".


Install Helm
Installing Rancher into our new Kubernetes cluster is easily done with Helm. Helm is a very popular package manager for Kubernetes. It is used as the installation tool for Rancher when deploying Rancher onto a Kubernetes cluster. In order to use Helm, we have to download the Helm CLI.

curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
  | bash
 Click to run on Rancher01
After a successful installation of Helm, we should check our installation to ensure that we are ready to install Rancher.

helm version --client
 Click to run on Rancher01
Helm uses the same kubeconfig as kubectl in the previous step.

We can check that this works by listing the Helm charts that are already installed in our cluster:

helm ls --all-namespaces
 Click to run on Rancher01


 Install cert-manager
cert-manager is a Kubernetes add-on to automate the management and issuance of TLS certificates from various issuing sources.

The following set of steps will install cert-manager which will be used to manage the TLS certificates for Rancher.

First, we'll add the helm repository for Jetstack

helm repo add jetstack https://charts.jetstack.io
 Click to run on Rancher01
Now, we can install cert-manager:

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.11.0 \
  --set installCRDs=true \
  --create-namespace
 Click to run on Rancher01
Once the helm chart has installed, you can monitor the rollout status of both cert-manager and cert-manager-webhook

kubectl -n cert-manager rollout status deploy/cert-manager
 Click to run on Rancher01
You should eventually receive output similar to:

Waiting for deployment "cert-manager" rollout to finish: 0 of 1 updated replicas are available...

deployment "cert-manager" successfully rolled out

kubectl -n cert-manager rollout status deploy/cert-manager-webhook
 Click to run on Rancher01
You should eventually receive output similar to:

Waiting for deployment "cert-manager-webhook" rollout to finish: 0 of 1 updated replicas are available...

deployment "cert-manager-webhook" successfully rolled out



Install Rancher
We will now install Rancher in HA mode onto our Rancher01 Kubernetes cluster. The following command will add rancher-latest as a helm repository.

helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
 Click to run on Rancher01
Finally, we can install Rancher using our helm install command.

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.34.244.231.92.sslip.io \
  --set replicas=1 \
  --version 2.7.4 \
  --create-namespace
 Click to run on Rancher01


 Verify Rancher is Ready to Access
Before we access Rancher, we need to make sure that cert-manager has signed a certificate using the dynamiclistener-ca in order to make sure our connection to Rancher does not get interrupted. The following bash script will check for the certificate we are looking for.

while true; do curl -kv https://rancher.34.244.231.92.sslip.io 2>&1 | grep -q "dynamiclistener-ca"; if [ $? != 0 ]; then echo "Rancher isn't ready yet"; sleep 5; continue; fi; break; done; echo "Rancher is Ready";
 Click to run on Rancher01



 Accessing Rancher
Note: Rancher may not immediately be available at the link below, as it may be starting up still. Please continue to refresh until Rancher is available.

Access Rancher Server at https://rancher.34.244.231.92.sslip.io.
For this Rodeo, Rancher is installed with a self-signed certificate from a CA that is not automatically trusted by your browser. Because of this, you will see a certificate warning in your browser. You can safely skip this warning. Some Chromium based browsers may not show a skip button. If this is the case, just click anywhere on the error page and type "thisisunsafe" (without quotes). This will force the browser to bypass the warning and accept the certificate.
Please follow instructions on UI to generate password for default admin user when prompted.
Make sure to agree to the Terms & Conditions
When prompted, the Rancher Server URL should be rancher.34.244.231.92.sslip.io, which is the hostname you used to access the server.
You will see the Rancher UI, with the local cluster in it. The local cluster is the cluster where Rancher itself runs, and should not be used for deploying your demo workloads.

In the top left corner of the UI, you can find a "burger menu" button, which opens up the global navigation menu. There you can access global applications and settings. You have quick links to explore all Rancher managed clusters and a way to get back to the Rancher home page.


Creating a Kubernetes Lab Cluster within Rancher
In this step, we will be creating a Kubernetes Lab environment within Rancher. Normally, in a production case, you would create a Kubernetes Cluster with multiple nodes; however, with this lab environment, we will only be using one virtual machine for the cluster.

Go back to the Rancher Home Page
On top of the list of available clusters, click Create
We will be using RKE2 cluster, so make sure to switch the toggle to RKE2/K3s
Note the multiple types of Kubernetes cluster Rancher supports. We will be using Custom cluster on existing nodes for this lab, but there are a lot of possibilities with Rancher.
Click on the Custom Cluster box in the Use existing nodes and create a cluster using RKE2 section
Enter a name in the Cluster Name box
Set the Kubernetes Version to a v1.24.x version
All other settings can be kept as default
Click Create at the bottom.
Once the cluster is created, you can retrieve an installation command in the Registration tab that you can use to add new nodes to your Kubernetes cluster.
Make sure the boxes etcd, Control Plane, and Worker are all ticked.
Click Show Advanced to the bottom right of the checkboxes
Enter the Node Public IP (54.171.242.44) and Node Private IP (172.31.47.123)
IMPORTANT: It is VERY important that you use the correct External and Internal addresses from the Cluster01 machine for this step, and run it on the correct machine. Failure to do this will cause the future steps to fail.
Check the checkbox to skip the TLS verification and accept insecure certificates below the registration command.
Double-click the registration command to copy it to your clipboard.
Proceed to the next step of this scenario.


Start the Rancher Kubernetes Cluster Bootstrapping Process
IMPORTANT NOTE: Make sure you have selected the Cluster01 tab in HobbyFarm in the window to the right. If you run this command on Rancher01 you will cause problems for your scenario session.

Take the copied command and run it on Cluster01
You can follow the provisioning process in the Machine Pools, Conditions and Related Resources tabs
Your cluster state in the cluster list and on the cluster detail page will change to Active
Once your cluster has gone to Active you can start exploring it by either clicking the Explore button in the cluster list on the home page, or by selecting the cluster in the global menu.


Interacting with the Kubernetes Cluster
In this step, we will be showing basic interaction with our Kubernetes cluster.

Click into your newly active cluster.
Note the diagrams dials, which illustrate cluster capacity, and the box that show you the recent events in your cluster.
Click the Kubectl Shell button (the button with the Prompt icon) in the top right corner of the Cluster Explorer, and enter kubectl get pods --all-namespaces and observe the fact that you can interact with your Kubernetes cluster using kubectl.
Also take note of the Download Kubeconfig File button next to it which will generate a Kubeconfig file that can be used from your local desktop or within your deployment pipelines.
In the left menu, you have access to all Kubernetes resources, the Rancher Application Marketplace and additional cluster tools.


Enable Rancher Monitoring
To deploy the Rancher Monitoring feature:

Navigate to Apps & Marketplace. in the left menu
Under Charts Locate the Monitoring chart, and click on it
On the Monitoring App Detail page click the Install button in the top right
This leads you to the installation wizard. In the first Metadata step, we can leave everything as default and click Next.
In the Values step, select the Prometheus section on the left. Change Resource Limits > Requested CPU from 750m to 250m and Requested Memory from 750Mi to 250Mi. This is required because our scenario virtual machine has limited CPU and memory available.
Click "Install" at the bottom of the page, and wait for the helm install operation to complete.
Once Monitoring has been installed, you can click on that application under "Installed Apps" to view the various resources that were deployed.

Working with Rancher Monitoring
Once Rancher Monitoring has been deployed, we can view the various components and interact with them.

In the left menu of the Cluster Explorer, select "Monitoring"
On the Monitoring Dashboard page, identify the "Grafana" link. Clicking this will proxy you to the installed Grafana server
Once you have opened Grafana, feel free to explore the various dashboard and visualizations that have been set up by default.

These options can be customized (metrics and graphs), but doing so is out of the scope of this scenario.

You will also see new Metrics and Alerts section on the Cluster page as well as on the individual workload pages.

Create a Deployment And Service
In this step, we will be creating a Kubernetes Deployment and Kubernetes Service for an arbitrary workload. For the purposes of this lab, we will be using the container image rancher/hello-world:latest but you can use your own container image if you have one for testing.

When we deploy our container in a pod, we probably want to make sure it stays running in case of failure or other disruption. Pods by nature will not be replaced when they terminate, so for a web service or something we intend to be always running, we should use a Deployment.

The deployment is a factory for pods, so you'll notice a lot of similarities with the Pod's spec. When a deployment is created, it first creates a replica set, which in turn creates pod objects, and then continues to supervise those pods in case one or more fails.

Under the Workloads sections in the left menu, go to Deployments and press Create in the top right corner and enter the following criteria:
Name - helloworld
Replicas - 2
Container Image - rancher/hello-world:latest
Under Ports click Add Port
Under Service Type choose to create a Node Port service
Enter 80 for the Private Container Port
NOTE: Note the other capabilities you have for deploying your container. We won't be covering these in this Rodeo, but you have plenty of capabilities here.
Scroll down and click Create
You should see a new helloworld deployment. If you click on it, you will see two Pods getting deployed.
From here you can click on a Pod, to have a look at the Pod's events. In the three-dots menu on a Pod, you can also access the logs of a Pod or start an interactive shell into the Pod.
In the left menu under Service Discovery > Services, you will find a new Node Port Service which exposes the hello world application publicly on a high port on every worker node. You can click on the linked Port to directly access it.


Create a Kubernetes Ingress
In this step, we will be creating a Layer 7 ingress to access the workload we just deployed in the previous step. For this example, we will be using sslip.io as a way to provide a DNS hostname for our workload. Rancher will automatically generate a corresponding workload IP.

In the left menu under Service Discovery go to Ingresses and click on *Create.
Enter the following criteria:
Name - helloworld
Request Host - helloworld.54.171.242.44.sslip.io
Path Prefix - /
Target Service - Choose the helloworld-nodeport service from the dropdown
Port - Choose port 80 from the dropdown
Click Create and wait for the helloworld.54.171.242.44.sslip.io hostname to register, you should see the rule become Active within a few minutes.
Click on the hostname and browse to the workload.
Note: You may receive transient 404/502/503 errors while the workload stabilizes. This is due to the fact that we did not set a proper readiness probe on the workload, so Kubernetes is simply assuming the workload is healthy.


Creating Projects in your Kubernetes Cluster
A project is a grouping of one or more Kubernetes namespaces. In this step, we will create an example project and use it to deploy a stateless WordPress.

In the left menu go to Cluster > Projects/Namespaces
Click Create Project in the top right
Give your project a name, like stateless-wordpress
Note the ability to add members, set resource quotas and a pod security policy for this project.
Next create a new namespace in the stateless-wordpress project. In the list of all Projects/Namespaces, scroll down to the stateless-wordpress project and click the Create Namespace button.
Enter the Name stateless-wordpress and click Create.


Add a new chart repository
The easiest way to install a complete WordPress into our cluster, is through the built-in Apps Marketplace. In addition to the Rancher and partner provided apps that are already available. You can add any other Helm repository and allow the installation of the Helm charts in there through the Rancher UI.

In the left menu go to Apps & Marketplace > Chart repositories
Click on Create in the top right
Enter the following details:
Name - rodeo
Target - Should be http(s) URL
Index URL - https://rancher.github.io/rodeo
Click on Create
Once the repository has been synchronized, go to Apps & Marketplace > Charts. There you will now see several new apps that you can install.


Deploy a Wordpress as a Stateless Application
In this step, we will be deploying WordPress as a stateless application in the Kubernetes cluster.

From Apps & Marketplace > Charts install the WordPress app
In step 1 of the installation wizard, choose the stateless-wordpress namespace and give the installation the name wordpress
In step 2 of the installation wizard, set:
WordPress' settings > WordPress password - to a password of your choice
Services and Load Balancing > Hostname - wordpress.54.171.242.44.sslip.io
Scroll to the bottom and click Install.
Once the installation is complete, navigate to Service Discovery > Ingresses. There you will see a new Ingress. Click on the URL to access WordPress.
Note: You may receive 404, 502, or 503 errors while the WordPress app is coming up. Simply refresh the page occasionally until WordPress is available
Log into WordPress using your set admin credentials - http://wordpress.54.171.242.44.sslip.io/wp-admin, and create a new blog post. Note that if you delete the wordpress-mariadb-0 pod or click Redeploy on the wordpress-mariadb StatefulSet you will lose your post. This is because there is no persistent storage under the WordPress MariaDB StatefulSet.

Deploy the nfs-server-provisioner into your Kubernetes Cluster
In a Kubernetes Cluster, it can be desirable to have persistent storage available for applications to use. As we do not have a Kubernetes Cloud Provider enabled in this cluster, we will be deploying the nfs-server-provisioner which will run an NFS server inside our Kubernetes cluster for persistent storage. This is not a production-ready solution by any means, but helps to illustrate the persistent storage constructs.

From Apps & Marketplace > Charts install the nfs-server-provisioner app
In step 1 of the installation wizard, choose the kube-system namespace and give the installation the name nfs-server-provisioner
In step 2 of the installation wizard, you can keep all the settings as default.
Scroll to the bottom and click Install.
Once the app is installed, go to Storage > Storage Classes
Observe the nfs storage class and the checkmark next to it which indicates it is the Default storage class.


Creating a Stateful WordPress Project in your Kubernetes Cluster
Let's deploy a second WordPress instance into the cluster that uses the NFS storage provider. First create a new project for it:

In the left menu go to Cluster > Projects/Namespaces
Click Create Project in the top right
Give your project a name, like stateful-wordpress
Note the ability to add members, set resource quotas and a pod security policy for this project.
Next create a new namespace in the stateful-wordpress project. In the list of all Projects/Namespaces, scroll down to the stateful-wordpress project and click the Create Namespace button.
Enter the Name stateful-wordpress and click Create.


Deploy WordPress as a Stateful Application
In this step, we will be deploying WordPress as a stateful application in the Kubernetes cluster. This WordPress deployment will utilize the NFS storage we deployed in the previous step to store our mariadb data persistently.

From Apps & Marketplace > Charts install the WordPress app
In step 1 of the installation wizard, choose the stateful-wordpress namespace and give the installation the name wordpress
In step 2 of the installation wizard, set:
WordPress' settings > WordPress password - to a password of your choice
Enable WordPress setting > WordPress Persistent Volume Enabled
Enable Database setting > MariaDB Persistent Volume Enabled
Services and Load Balancing > Hostname - stateful-wordpress.54.171.242.44.sslip.io
Scroll to the bottom and click Install.
Once the installation is complete, navigate to Service Discovery > Ingresses. There you will see a new Ingress. Click on the URL to access WordPress.
Note: You may receive 404, 502, or 503 errors while the WordPress app is coming up. Simply refresh the page occasionally until WordPress is available
Note that you now have two Persistent Volumes available under Storage > Persistent Volumes
Log into WordPress using your set admin credentials - http://stateful-wordpress.54.171.242.44.sslip.io/wp-admin and create a new blog post. If you delete the wordpress-mariadb pod or click Redeploy now, your post will not be lost.


Upgrading your Kubernetes Cluster
This step shows how easy it is to upgrade your Kubernetes clusters within Rancher.

In the global menu go to Cluster Management
Click the ellipses button (...) at the end of the line of the second cluster you created
Click Edit Config
Under Kubernetes Version select a newer version of Kubernetes
Scroll down and hit Save
Observe that your Kubernetes cluster will now be upgraded.


Congratulations
Congratulations, you have finished the Scenario. If you would like to tear down your lab environment, you can click the "Finish" button, otherwise, continue to work with your Kubernetes cluster while keeping HobbyFarm in the background.

