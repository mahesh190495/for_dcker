 

Introduction
------------

Modern microservices-based applications that are deployed into Kubernetes often consist of tens or hundreds of services. The resource constraints and a number of these services mean that it is often difficult or impossible to run all of this on a local development machine, which makes fast development and debugging very challenging.

[Telepresence](https://www.getambassador.io/docs/telepresence) enables you to connect your local development machine seamlessly to the K8s cluster via a two-way proxying mechanism. This enables you to code locally and run the majority of your services within a K8s cluster.

This document provides walkthroughs to utilize telepresence in local development.

Before you begin
----------------

We need a K8s cluster to follow the workflow demonstrations. We recommend minikube for creating the K8s cluster on your workstation

### K8s cluster setup using minikube

1.  [Install minikube](https://minikube.sigs.k8s.io/docs/start/)
2.  To create a minikube cluster run
    
    minikube start
    
      
    

### Source code for the examples used in the walkthrough.

To follow along with the walkthrough with the same examples showcased in upcoming sections, please download the [source code](https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1)

\# you can clone/download the source code in the following ways:
1)Go to https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1 (will be changed after merge of pull branch) -> Code -> Download zip -> extract the code from zip  
(In this case the souce code will be in folder "iatcp-telepresence-maheshp-telepresence-examples-v1" )

2) $ git clone https://github.mathworks.com/development/iatcp-telepresence-maheshp.git -b telepresence-examples-v1
(In this case the souce code will be in folder "iatcp-telepresence-maheshp" )

  

Installing telepresence
-----------------------

### Installing telepresence traffic manager on your K8s cluster

We will be using Helm to install the telepresence traffic manager on your K8s cluster.

If you do not have the helm, you can install it from [here](https://helm.sh/docs/intro/install/). Next, follow the below steps

1.  Add the Required helm repository
    
    $ helm repo add mw-helm http://mw-helm-repository.mathworks.com/artifactory/mw-helm "mw-helm" has been added to your repositories
    
      
    
2.  Create ambassador namespace
    
    $ kubectl create namespace ambassador 
    namespace/ambassador created 
    
      
    
3.  Run the below commands to deploy the telepresence traffic manager to your K8s cluster.
    
    #Obtain the values.yaml file
    #You can find the values.yaml file in the directory iatcp-telepresence-maheshp-telepresence-examples-v1/traffic-manager-helm-values/values.yaml, if you have downloaded the source code. If not run the below command with your access token
    $curl https://github.mathworks.com/raw/development/iatcp-telepresence-maheshp/telepresence-examples-v1/traffic-manager-helm-values/values.yaml?token=<your access token> >values.yaml
    
    
    #The below command deploys the telepresence traffic manager with configurations in values.yaml
    $ helm install traffic-manager --namespace ambassador mw-helm/telepresence --version 2.10.4 -f values.yaml
    NAME: traffic-manager
    LAST DEPLOYED: Tue Apr  4 15:05:53 2023
    NAMESPACE: ambassador
    STATUS: deployed
    REVISION: 1
    NOTES:
    --------------------------------------------------------------------------------
    Congratulations!
    
    
    You have successfully installed the Traffic Manager component of Telepresence!
    Now your users will be able to \`telepresence connect\` to this Cluster and create
    intercepts for their services!
    
      
    

  

### Installing telepresence daemon on the workstation

Install Telepresence by running the respective commands below for your OS

#### Linux

\# 1.Copy telepresence binary from //mathworks/hub/3rdparty/internal/9506312/glnxa64/Telepresence/ to any buffer directory (the binary will be moved to actual path in 3rd step)
$ cp //mathworks/hub/3rdparty/internal/9506312/glnxa64/Telepresence/telepresence ~/Downloads

# 2.Make it an executable
$ chmod a+x ~/Downloads/telepresence
  
# 3.Move the file to /usr/local/bin 
$ sudo mv ~/Downloads/telepresence /usr/local/bin/

  

#### macOS

\# Intel Macs

# 1. Copy telepresence binary from //mathworks/hub/3rdparty/internal/9506312/maci64/Telepresence to any buffer directory (the binary will be moved to actual path in next step)
$ cp //mathworks/hub/3rdparty/internal/9506312/maci64/Telepresence/telepresence ~/Downloads

# 2.Move the file to /usr/local/bin
$sudo mv ~/Downloads/telepresence /usr/local/bin

# Apple silicon Macs
# 1. Copy telepresence binary from //mathworks/hub/3rdparty/internal/9506312/maca64/Telepresence to any buffer directory (the binary will be moved to actual path in next step)
$ cp //mathworks/hub/3rdparty/internal/9506312/maca64/Telepresence/telepresence ~/Downloads

# 2.Move the file to /usr/local/bin
$sudo mv ~/Downloads/telepresence /usr/local/bin 

  

#### Windows

\# To install Telepresence, run the following commands in PowerShell as admin

# 1. Copy telepresence zip file from 3rdparty/internal/9506312/win64/Telepresence to any buffer directory (the binary will be moved to actual path in 3rd step)
PS C:\\Users\\maheshp> cp \\\\mathworks\\BGL\\hub\\3rdparty\\internal\\9506312\\win64\\Telepresence\\telepresence.zip .\\Downloads\\


# 2.Navigate to the folder containing the downloaded file and  unzip the telepresence.zip file to the desired directory, then remove the zip file
PS C:\\Users\\maheshp\\Downloads> Expand-Archive -Path telepresence.zip -DestinationPath telepresenceInstaller/telepresence 
PS C:\\Users\\maheshp\\Downloads>Remove-Item 'telepresence.zip'
PS C:\\Users\\maheshp\\Downloads> cd .\\telepresenceInstaller\\telepresence\\


# 3. Run the install-telepresence.ps1 to install telepresence's dependencies. It will install telepresence to
# C:\\telepresence by default, but you can specify a custom path by passing in -Path C:\\my\\custom\\path 
PS C:\\Users\\maheshp\\Downloads\\telepresencInstaller\\telepresence\\> powershell.exe -ExecutionPolicy bypass -c " . '.\\install-telepresence.ps1';"

# 4. Remove the unzipped directory:  
PS C:\\Users\\maheshp\\Downloads\\telepresencInstaller\\telepresence\\> cd ../.. 
PS C:\\Users\\maheshp\\Downloads>Remove-Item telepresenceInstaller -Recurse -Confirm:$false -Force

# 5. Telepresence is now installed and you can use telepresence commands in PowerShell

  

**Workflow 1: Using telepresence to develop code on the K8s cluster from your workstation**
-------------------------------------------------------------------------------------------

Before we proceed let's take a look at some pre-requisites

1.  Telepresence can be used to intercept the traffic from the following types of K8s workload
    1.  [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
    2.  [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
    3.  [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
2.  The Workload should have a [K8s service](https://kubernetes.io/docs/concepts/services-networking/service/) associated with it (ClusterIP, NodePort, LoadBalancer)
3.  The current context in your config file in the .kube folder should point to the K8s cluster where you want to use telepresence.
4.  If you want to avoid downtime of the microservice you are trying to intercept, then make sure the code of the microservice is running on the workstation

In the following walkthrough, we will use an HTTP server written in Golang which serves an HTML static file.

To set this up, download [](https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1/example1)the source code as mentioned [here](https://confluence.mathworks.com/display/IATCP/User+document%3A+Telepresence+workflows#Userdocument:Telepresenceworkflows-Sourcecodefortheexamplesusedinthewalkthrough.) and navigate to iatcp-telepresence-maheshp-telepresence-examples-v1/example1/

\[maheshp@vdi-dd1bgl-022:~/Desktop\] ...
$ ls
iatcp-telepresence-maheshp-telepresence-examples-v1/

$ cd iatcp-telepresence-maheshp-telepresence-examples-v1/example1/

  

### Steps to create the example1 one deployment on K8s cluster

  

Steps

Details

1

First, we will create the container image, navigate to the example1-image folder and build the docker image (pointing to the localhost registry) and load it to minikube

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example1

#Navigate to example1-image
$ cd example1-image/

# Building the image 
$ docker build -t example1 .

#Load the image to minikube
$ minikube image load example1

#Navigate back to application directory
$cd ..

  

2

Create the deployment using the "my-microservice" helm chart in the directory example1/helm/

  

#Navigate to helm directory
$ cd helm/

$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example1/helm


#helm install <name of release>
$ helm install mahesh my-microservice

#Navigate back to example1 folder
$ cd ..


#Once the Deployments are ready we can launch the service in browser with minikube service <service name>
$ minikube service mahesh-my-microservice

  

3

We can see that there is a hello message from the K8s cluster is being served at the K8s node port.

  

#Once the Deployments are ready we can launch the service in browser with "minikube service <service name>" 
$ minikube service mahesh-my-microservice
|-----------|------------------------|-------------|---------------------------|
| NAMESPACE |          NAME          | TARGET PORT |            URL            |
|-----------|------------------------|-------------|---------------------------|
| default   | mahesh-my-microservice | http/5000   | http://192.168.49.2:30260 |
|-----------|------------------------|-------------|---------------------------|
🎉  Opening service default/mahesh-my-microservice in default browser...

#This will provide <the Node-IP>:<Node-Port> URL of the K8s cluster and launch the same in your default browser.

![](/download/attachments/883387724/image2023-2-27_19-29-24.png?version=1&modificationDate=1677506365000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-2-27_19-29-24.png")

  

### Steps to run the example1 code on your workstation

  

Steps

Details

1

Now to run the code locally make sure you have Golang installed, you can do it from [here](https://mathworks.sharepoint.com/sites/devu/go/SitePages/Setup-the-development-environment-in-Go.aspx?ga=1#install-golang)

  

2

Run the example1/main.go file.

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example1

$ ls
example1-image/  go.mod  helm/  main.go  readme-example1  static/

# Run the golang http server
  go run main.go

  

3

The index.html page in the static folder in the application directory will now be served at the workstation port 5000

![](/download/attachments/883387724/image2023-2-27_19-37-37.png?version=1&modificationDate=1677506858000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-2-27_19-37-37.png")

### Steps to use Telepresence for developing the code live on the cluster

  

Steps

Details

1

Create the my-microservice deployment in your minikube cluster as mentioned [here](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs?focusedCommentId=929454596#TelepresenceWalkthroughs-Stepstocreatetheexample1onedeploymentonK8scluster)

  

$ kubectl get deployment
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
mahesh-my-microservice   1/1     1            1           5m39s

  

2

Connect to telepresence to the minikube cluster.

  

#This will connect the telepresence daemon on your workstation with telepresence traffic manager on the K8s cluster.
$ telepresence connect
Connected to context minikube (https://192.168.49.2:8443)

  

3

You can then list the workloads available for intercepting

  

$ telepresence list 
mahesh-my-microservice: ready to intercept (traffic-agent not yet installed)

  

4

You can intercept the workload with this command

  

Now all the requests to my-microservice are being routed to port 5000 on my workstation.

  

#telepresence intercept <name of workload> --port <port on workstation where your code is listening>   
$ telepresence intercept mahesh-my-microservice --port 5000
Using Deployment mahesh-my-microservice
intercepted
   Intercept name         : mahesh-my-microservice
   State                  : ACTIVE
   Workload kind          : Deployment
   Destination            : 127.0.0.1:5000
   Service Port Identifier: http
   Volume Mount Point     : /tmp/telfs-1428258977
   Intercepting           : all TCP requests

  

5

Run the Golang main.go file in the app directory(example1/main.go)

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example11

$ ls
example1-image/  go.mod  helm/  main.go  readme-example1  static/

# Run the golang http server
  go run main.go

  

6

We can see that the request at the K8s cluster's end( 192.168.49.2:31814 which is the minikube cluster IP) is being served by the code running on the workstation.

  

![](/download/attachments/883387724/image2023-4-5_13-51-48.png?version=1&modificationDate=1680682915000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-5_13-51-48.png")

  

We can see that the request at the K8s cluster's end (192.168.49.2:31814 which is the minikube cluster IP) is being served by the code running on the workstation.

You can check your K8s cluster ip on for minkube with below command
$ minikube profile list
🎉  minikube 1.30.1 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.30.1
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

|----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| Profile  | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| minikube | docker    | docker  | 192.168.49.2 | 8443 | v1.23.0 | Running |     1 | \*      |
|----------|-----------|---------|--------------|------|---------|---------|-------|--------|

#OR

#Once the Deployments are ready we can launch the service in browser with "minikube service <service name>" 
$ minikube service mahesh-my-microservice
|-----------|------------------------|-------------|---------------------------|
| NAMESPACE |          NAME          | TARGET PORT |            URL            |
|-----------|------------------------|-------------|---------------------------|
| default   | mahesh-my-microservice | http/5000   | http://192.168.49.2:30260 |
|-----------|------------------------|-------------|---------------------------|
🎉  Opening service default/mahesh-my-microservice in default browser...

#This will provide <the Node-IP>:<Node-Port> URL of the K8s cluster and launch the same in your default browser.

  

7

Now we are good to code on the K8s cluster from the workstation. Make any changes to the code running on your workstation and see results at the K8s cluster's end.

![](/download/attachments/883387724/image2023-3-17_17-11-40.png?version=1&modificationDate=1679053301000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-17_17-11-40.png")

You can use any editor, or any tool you usually use when developing locally on your workstation to update the code, and the results will reflect at the K8s cluster's end.

8

**Similarly, users can do multiple changes to their code on their workstation and check the results at the K8s cluster's end without having to, build the image, upload it to the repo and deploy it to see the result at the K8s cluster's end every time. This reduces the iterations of the [inner development loop](https://www.getambassador.io/docs/telepresence/latest/concepts/devloop#what-is-the-inner-dev-loop).**

  

  

9

Once you are done intercepting you can stop the intercept and disconnect the telepresence.

  

#Run the below command to stop the intercept
$ telepresence leave mahesh-my-microservice

#Run the below command to disconnect  telepresence 
$ telepresence quit
Telepresence Daemons disconnecting...done

  

10

To clean up the K8s cluster we can use Helm to uninstall the deployment

  

$ helm uninstall mahesh
release "mahesh" uninstalled

  

**Workflow 2: Using telepresence to set up your local development workspace**
-----------------------------------------------------------------------------

We will now see how users can use telepresence to extract env variables and volume mounts to mimic the K8s cluster.

The example microservice serves as a static HTML page (in volume mount of the k8s deployment) that will display the value of the env variable defined inside the K8s cluster (at the container level).

To set this up, download [](https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1/example1)the source code as mentioned [here](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs#TelepresenceWalkthroughs-Sourcecodefortheexamplesusedinthewalkthrough.) and navigate to iatcp-telepresence-maheshp-telepresence-examples-v1/example2/

\[maheshp@vdi-dd1bgl-022:~/Desktop\] ...
$ ls
iatcp-telepresence-maheshp-telepresence-examples-v1/

$ cd iatcp-telepresence-maheshp-telepresence-examples-v1/example2/

### Steps to create the example2 deployment on K8s cluster

  

Steps

Details

1

First, we will create the container image, and navigate to the example2/example2-image folder.

Build the docker image (pointing to the localhost registry) and load it to minikube.

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example2

#Navigate to example2-image
$ cd example2-image/

# Building the image 
$ docker build -t example2 .   

#Load the image to minikube
$ minikube image load example2

#Navigate back to application folder
$ cd ..

  

2

This static file will be provided to the deployment via a volume mount(host path), hence first we will create index.html in our K8s node

  

3

SSH to the K8s node

  

#We are using Minkube for creating the K8s cluster with container driver as docker.The below command will ssh to minikube #node
$ minikube ssh
docker@minikube:~$

  

4

Create a folder and home.html that will be used in the volume mount.

  

#creating the folder in the mount path 
docker@minikube:~$ sudo mkdir /mnt/data 

#move to the data folder
docker@minikube:~$cd /mnt/data  
docker@minikube:/mnt/data$ sudo vi home.html

#Add the below html code and save
<!DOCTYPE html>
<html>
	<head>
		<meta charset="UTF-8">
		<title>Environment variable values declared at container level</title>
	</head>
	<body>
		<h1>Environment Variable Values:</h1>
		<ul>
			<li><strong>{{ .EnvVariableName }} = </strong> {{ .EnvVariableValue }}</li><!-- Display the value of the environment variable here -->
		</ul>
	</body>
</html>

#Exit the node
docker@minikube:~$ exit
logout

**Note if using Windows**: If you are using Windows and have trouble creating the home.html in the minikube node, exit the node and create home.html #in your local and then run minikube cp ./home.html [minikube:/mnt/data/home.html](http://minikube/mnt/data/home.html)

5

Create the deployment using the "my-microservice" helm chart in example2/helm directory

  

#Navigate to helm directory
$ cd helm/

$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example2/helm

#helm install <name of release>
$ helm install mahesh my-microservice

#Once the Deployments are ready we can launch the service in browser with minikube service <service name>
$ minikube service mahesh-my-microservice

  

6

We can see that the home.html in the volume mount is now being served with environment variables at the K8s cluster level like KUBERNETETETES\_SERVICE\_HOST AND KUBERNETETETES\_SERVICE\_PORT, at the K8s Cluster node port.

  

#Once the Deployments are ready we can launch the service in browser with "minikube service <service name>" 
$ minikube service mahesh-my-microservice
|-----------|------------------------|-------------|---------------------------|
| NAMESPACE |          NAME          | TARGET PORT |            URL            |
|-----------|------------------------|-------------|---------------------------|
| default   | mahesh-my-microservice | http/5000   | http://192.168.49.2:30260 |
|-----------|------------------------|-------------|---------------------------|
🎉  Opening service default/mahesh-my-microservice in default browser...

#This will provide <the Node-IP>:<Node-Port> URL of the K8s cluster and launch the same in your default browser.

  

![](/download/attachments/883387724/image2023-3-19_23-15-8.png?version=1&modificationDate=1679247909000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-19_23-15-8.png")

### Steps to use telepresence for setting up your local development workspace

  

Steps

Details

1

Make sure you have the latest deployment of the microservice that you want to work on. In our example case, we have created a deployment called mahesh-my-microservice as explained [previously](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs#TelepresenceWalkthroughs-Stepstocreatetheexample2deployment).

  

$ kubectl get deployment
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
mahesh-my-microservice   1/1     1            1           5m39s

  

2

Connect telepresence to your K8s cluster, telepresence will use the config file in the ./kube folder to identify the cluster to which it will connect.

  

#This will connect telepresence daemon on your workstation with telepresence traffic manager on the K8s cluster
$ telepresence connect
Connected to context minikube (https://192.168.49.2:8443)

#This will list all the workloads we can intercept
$ telepresence list 
mahesh-my-microservicemy-microservice: ready to intercept (traffic-agent not yet installed)

#Currently we are in application directory
$ pwd
/home/..../Telepresence-examples-main/example2/

#We will create a folder to store all the different volume mounts from the k8s deployment
$ mkdir local\_volume\_mount

#Run below command to intercept the miroservice
#--port is the port on your workstation where all the traffic from  the K8s cluster will be #routed to.
#--env-file will create a .env file in the specified path and name with all the envoirment #variables in #at the container level.
#--mount will copy all the volume mounts of worklaod to the path specified.

$ telepresence intercept mahesh-my-microservice --port 5000 --env-file ./local.env --mount ./local\_volume\_mount/
Using Deployment mahesh-my-microservice
intercepted
   Intercept name         : mahesh-my-microservice
   State                  : ACTIVE
   Workload kind          : Deployment
   Destination            : 127.0.0.1:5000
   Service Port Identifier: http
   Volume Mount Point     : /home/..../Telepresence-examples-main/example2/local\_volume\_mount
   Intercepting           : all TCP requests  

#NOTE:Windows can only perform remote mounts using drives,and not folders hence we need to provide a Letter for #drive(unused) followed by a colon i.e --mount=X:, Therefore for windows user the command will look like this
PS C:\\Users\\maheshp\\Desktop\\example2> telepresence intercept mahesh-my-microservice --port 5000 --env-file=.\\local.env --mount X:
Using Deployment mahesh-my-microservice
intercepted
   Intercept name         : mahesh-my-microservice
   State                  : ACTIVE
   Workload kind          : Deployment
   Destination            : 127.0.0.1:5001
   Service Port Identifier: http
   Volume Mount Point     : X:
   Intercepting           : all TCP requests

  

3

We can see that all the directories in volume mount(/app/static in the container) have been replicated to the local\_volume\_mount directory and all the environment variables(at container level) have been copied to local.env

![](/download/attachments/883387724/image2023-3-19_23-19-29.png?version=1&modificationDate=1679248169000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-19_23-19-29.png")

4

Now we can incorporate the static file in local\_volume\_mount and the env variables from local .env in the code running on our workstation to mimic the k8s cluster.

Here in our code running locally (iatcp-telepresence-maheshp-telepresence-examples-v1/example2/main.go ) we can see that we load and use the env variables from local.env(which has env values from the K8s cluster). Also, we serve the HTML page in the local\_volume\_mount (which is the replicated volume).

  

  

#We can see in line 21 and 26 of the code that we are loading and using the environment variables in the K8s cluster,that is from #local.env 

#line 21
$ sed -n 21p main.go 
        err := godotenv.Load("local.env")
#line 26
$ sed -n 26p main.go 
        envValue := os.Getenv("KUBERNETES\_SERVICE\_HOST") + ":" + os.Getenv("KUBERNETES\_SERVICE\_PORT") err := 

#Serving the static file downaloaded at local\_volume\_mount which we replicated from the K8s cluster
#The below line is for linux/mac machines
tmpl, err := template.ParseFiles("./local\_volume\_mount/app/static/home.html")

**Note for Windows:** Since the Directory specified for storing the replicated volume mount is different for Windows, please make the following edit:

Uncomment the below line and comment out the one for Linux in your example2/main.go file

  

tmpl, err :=template.ParseFiles("X:/app/static/home.html")

  

5

Run the go application on your workstation (example2/main.go)

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example2

$ ls
example2-image/  go.mod  go.sum  helm/  main.go

# Run the golang http server
  go run main.go

  

6

We can see from the image that the code running on our workstation is using the home.html, a file mounted in the K8s Container, and environment variables defined at the K8s cluster, to serve the request mimicking the results at the K8s cluster's end.

![](/download/attachments/883387724/image2023-3-19_23-31-29.png?version=1&modificationDate=1679248890000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-19_23-31-29.png")

7

**Similarly, users can obtain and use the same environment variable values and files in the volume mounts in their workstation as it is present in the K8s cluster.**

  

8

Once you are done intercepting you can stop the intercept and disconnect the telepresence.

  

#Run the below command to stop the intercept
$ telepresence leave mahesh-my-microservice

#Run the below command to disconnect  telepresence 
$ telepresence quit
Telepresence Daemons disconnecting...done

  

  

To clean up the K8s cluster we can use Helm to uninstall the deployment

  

$ helm uninstall mahesh
release "mahesh" uninstalled

  

  

  

  

**Workflow 3: Using telepresence to debug your code at the K8s cluster level**
------------------------------------------------------------------------------

Suppose there was a code change after which the microservice is not working as expected. This might be due to several reasons such as wrong logic, issues with the efficiency or reliability of code, or issue w.r.t connectivity with other k8s resources. These issues can be solved faster if we can use the same debugging tools and procedures we do while developing the microservice locally, at the K8s cluster level(which is isolated ).

Our example is a simple word count application that takes text as input and gives you an analysis of words, like the count of each word, and the total number of words.

To set this up, download [](https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1/example1)the source code as mentioned [here](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs#TelepresenceWalkthroughs-Sourcecodefortheexamplesusedinthewalkthrough.) and navigate to iatcp-telepresence-maheshp-telepresence-examples-v1/example2/

\[maheshp@vdi-dd1bgl-022:~/Desktop\] ...
$ ls
iatcp-telepresence-maheshp-telepresence-examples-v1/

$ cd iatcp-telepresence-maheshp-telepresence-examples-v1/example3/

### Steps to create the example3 one deployment on the K8s cluster

  

Steps

Details

1

First, we will create the container image, for this navigate to the example3/example3-image folder.

Build the docker image and load it to minikube

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example3

#Navigate to example3-image
$ cd example3-image/

# Building the image 
$ docker build -t example3 .   

#Load the image to minikube
$ minikube image load example3

#Navigate back to application directory
$ cd ..

  

2

Create the deployment using the "my-microservice" helm chart in the example3/helm folder

  

#Navigate inside helm directory
$ cd helm/

$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example3/helm

#helm install <name of release>
$ helm install mahesh my-microservice


#Navigate back to application directory
$ cd ..

  

3

We can see that the word counter project is now being served by the K8s cluster at the [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)

  

#Once the Deployments are ready we can launch the service in browser with "minikube service <service name>" 
$ minikube service mahesh-my-microservice
|-----------|------------------------|-------------|---------------------------|
| NAMESPACE |          NAME          | TARGET PORT |            URL            |
|-----------|------------------------|-------------|---------------------------|
| default   | mahesh-my-microservice | http/5000   | http://192.168.49.2:30260 |
|-----------|------------------------|-------------|---------------------------|
🎉  Opening service default/mahesh-my-microservice in default browser...

#This will provide <the Node-IP>:<Node-Port> URL and launch the same in your default browser.

  

![](/download/thumbnails/883387724/image2023-3-1_12-46-57.png?version=1&modificationDate=1677655018000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_12-46-57.png")

![](/download/thumbnails/883387724/image2023-3-1_12-47-34.png?version=1&modificationDate=1677655055000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_12-47-34.png")

  

### Steps to run the example3 code on your workstation

  

Steps

Details

1

Now to run the code locally make sure you have golang installed, you can do it from [here](https://mathworks.sharepoint.com/sites/devu/go/SitePages/Setup-the-development-environment-in-Go.aspx?ga=1#install-golang)

  

2

Navigate to example3/main.go file and run the main.go file.

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example3

$ ls
example3-image/  go.mod  helm/  main.go  readme-example3


# Run the golang http server
  go run main.go

  

4

The index.html page in the static folder in the application directory will now be served at the workstation port 5000 (i.e http://localhost:5000)

![](/download/attachments/883387724/image2023-3-1_12-59-57.png?version=1&modificationDate=1677655798000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_12-59-57.png")

### Steps to use telepresence to debug your code at the K8s cluster level

  

Steps

Details

1

Deploy the wordcount microservice on the K8s cluster and verify.

  

$ minikube service mahesh-my-microservice
|-----------|------------------------|-------------|---------------------------|
| NAMESPACE |          NAME          | TARGET PORT |            URL            |
|-----------|------------------------|-------------|---------------------------|
| default   | mahesh-my-microservice | http/5000   | http://192.168.49.2:30260 |
|-----------|------------------------|-------------|---------------------------|
🎉  Opening service default/mahesh-my-microservice in default browser...

  

2

We can observe that when we provide input in the text box and click on "CountWords", we get incorrect results

![](/download/attachments/883387724/image2023-3-1_13-3-29.png?version=1&modificationDate=1677656009000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_13-3-29.png")![](/download/attachments/883387724/image2023-3-1_13-3-50.png?version=1&modificationDate=1677656031000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_13-3-50.png")

3

Connect telepresence to the K8s cluster and intercept the microservice.

  

$ telepresence intercept mahesh-my-microservice --port 5000
Using Deployment mahesh-my-microservice
intercepted
   Intercept name         : mahesh-my-microservice
   State                  : ACTIVE
   Workload kind          : Deployment
   Destination            : 127.0.0.1:5000
   Service Port Identifier: http
   Volume Mount Point     : /tmp/telfs-761166714
   Intercepting           : all TCP requests

  

4

Run the same main.go file as we did [previously](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs#TelepresenceWalkthroughs-Stepstoruntheexample3codeonyourworkstation) but in VScode (or your preferred IDE/Debugger) in debug mode with relevant breakpoints.

  

![](/download/attachments/883387724/image2023-3-1_13-10-34.png?version=1&modificationDate=1677656436000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-1_13-10-34.png")

5

Now give some input and hit submit. And traverse via the code step by step to catch the bug. In the last image, we see that the issue, len((wordCounts) + 1) is being returned.

![](/download/attachments/883387724/image2023-4-6_11-54-47.png?version=1&modificationDate=1680762288000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-6_11-54-47.png")

6

You can edit the code to fix the issue(change it to len(words)) and save the file, reload the page, and test the fix again.

![](/download/attachments/883387724/image2023-4-6_12-1-10.png?version=1&modificationDate=1680762671000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-6_12-1-10.png")

7

**Similarly, you can debug, analyze, and update your microservices running on the K8s cluster locally on your workstation using any tool you normally would when during local development.**

  

8

Once you are done intercepting you can stop the intercept and disconnect the telepresence.

  

#Run the below command to stop the intercept
$ telepresence leave mahesh-my-microservice

#Run the below command to disconnect  telepresence 
$ telepresence quit
Telepresence Daemons disconnecting...done

  

  

To clean up the K8s cluster we can use Helm to uninstall the deployment

  

$ helm uninstall mahesh
release "mahesh" uninstalled

  

**Workflow 4: Using telepresence for automated testing of cloud-native code**
-----------------------------------------------------------------------------

Users can utilize the fact that all the endpoints within the K8s cluster are made available to the workstation once telepresence is connected to the K8s cluster. They can create scripts for automation/testing and also use tools like Postman, curl, netcat, nslookup on your workstation to test the microservice and the entire architecture from within the K8s cluster.

Let's consider an example where there are 2 versions of microservices that provide some processed data from their API endpoints which will be further used by the rest of the application workflow and you want to test this microservice live and compare the data returned by both the services.

To set this up, download [](https://github.mathworks.com/development/iatcp-telepresence-maheshp/tree/telepresence-examples-v1/example1)the source code as mentioned [here](https://confluence.mathworks.com/display/IATCP/Telepresence+Walkthroughs#TelepresenceWalkthroughs-Sourcecodefortheexamplesusedinthewalkthrough.) and navigate to iatcp-telepresence-maheshp-telepresence-examples-v1/example2/

\[maheshp@vdi-dd1bgl-022:~/Desktop\] ...
$ ls
iatcp-telepresence-maheshp-telepresence-examples-v1/

$ cd iatcp-telepresence-maheshp-telepresence-examples-v1/example4/

### Steps to create the example4 deployment on K8s cluster

  

Steps

Details

1

First, we will create the container images.

Navigate to the example4/example4-image folder.

Build the docker image of the first microservice and load it to minikube

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example4

#Navigate to example4-image
$ cd example4-image/

# Building the image
$ docker build -t example4:v1 .  

#Load the image to minikube
$ minikube image load example4:v1

  

2

Now let's create and update the microservice and the docker file to create a new version of the microservice (v2).

  

3

First edit the file example4/example4-image/main.go with the following changes:-

1.  Update the value of age for any user ( this will simulate the deviation in the data served by the new version)
2.  Update the API endpoint port.

  

1)![](/download/attachments/883387724/image2023-3-14_10-49-35.png?version=1&modificationDate=1678771176000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-3-14_10-49-35.png")

2)![](/download/attachments/883387724/image2023-4-4_19-1-40.png?version=1&modificationDate=1680615101000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-4_19-1-40.png")

4

Next, we will update the API endpoint to be exposed in the docker file as well

![](/download/attachments/883387724/image2023-4-4_19-3-9.png?version=1&modificationDate=1680615189000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-4_19-3-9.png")

5

Now we build and load the image of the updated microservice (v2)

  

\# Building the image
$ docker build -t example4:v2 .

#Load the image to minikube
$ minikube image load example4:v2

#Navigate back to application directory
$ cd ..

  

6

We can now deploy the 2 microservices app1-my-microservice and app2-my-microservice

To do this go to Telepresence-examples-main/example4/helm/ and run the mentioned commands

  

#Navigate to helm directory
$ cd helm/

$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example4/helm

#Installing first microservice
$ helm install app1 my-microservice

#Installing second microservice
$ helm install app2 my-microservice -f values2.yaml

#Naviagate back to the application folder
$ cd ..

  

7

Since the services of the deployments in the K8s cluster is of type ClusterIP, the endpoints are only exposed to the internal network of the K8s cluster, and hence  
the user cannot interact with the API endpoints or microservices from his workstation

  

### Steps to create a script to test the K8s workloads with telepresence

  

Steps

Details

1

Once the microservices are deployed to the K8s cluster the resources and endpoints of the microservice are isolated to the K8s network and are not accessible directly.

We can see that when we try to _curl_ or perform _nslookup_ the internal service urls

  

  

#These are the k8 services of the miroservices app1 and app2
$ kubectl get svc
 NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    
app1-my-mircoservice   ClusterIP   10.107.194.180   <none>        5000/TCP   
app2-my-mircoservice   ClusterIP   10.100.187.151   <none>        5001/TCP   




$ nc -zv  app1-my-microservice.default 8080
nc: getaddrinfo for host "app1-my-microservice.default" port 8080: Name or service not known   


$ curl http://app1-my-microservice.default:8080/users
curl: (6) Could not resolve host: app1-my-microservice.default   


$ nslookup app1-my-microservice.default
Server:         172.18.74.9
Address:        172.18.74.9#53

\*\* server can't find app1-my-microservice.default: NXDOMAIN

  

2

Connect telepresence to your K8s cluster

  

$ telepresence connect
Connected to context minikube (https://192.168.49.2:8443)

  

3

Once telepresence is connected to your K8s cluster your workstation behaves as part of the K8s internal network and hence all the resources are now accessible.

we can see that all the commands that previously failed now succeed

  

$ nslookup app1-my-microservice.default
Server:         172.18.74.9
Address:        172.18.74.9#53

Name:   app1-my-microservice.default
Address: 10.104.129.106  

$ curl http://app1-my-microservice.default:8080/users
\[{"name":"Alice","age":23},{"name":"Bob","age":30},{"name":"Charlie","age":35}\]   

$ nc -zv  app1-my-microservice.default 8080
Connection to app1-my-microservice.default (10.104.129.106) 8080 port \[tcp/http-alt\] succeeded!   

  

4

We can hit the service URLs directly to on the workstation browser as well

![](/download/attachments/883387724/image2023-4-6_14-22-11.png?version=1&modificationDate=1680771133000&api=v2 "IAT Cloud Platform > User document: Telepresence workflows > image2023-4-6_14-22-11.png")

5

We can run scripts (example4/main.go )from our workstation which refer to these internal endpoints directly

  

#Current directory
$ pwd
/home/maheshp/Desktop/iatcp-telepresence-maheshp-telepresence-examples-v1/example4

$ ls
example4-image/  helm/  main.go  readme-example4


# Run the golang http server
  go run main.go

  

  

6

Here we can see that the script creates GET requests to each microservice and then compares the responses and prints a relevant message if there is a deviation in the response

  

#Below code snippet is from the main.go script in the previous step.
#We can see that the code executes a get request to the API endpoint which is only accesible from within the K8s cluster



resp, err := http.Get("http://app1-my-microservice.default:8080/users")
	if err != nil {
		fmt.Println("Error: ", err)
		return
	}
	defer resp.Body.Close()

resp1, err := http.Get("http://app2-my-microservice.default:8081/users")
	if err != nil {
		fmt.Println("Error: ", err)
		return
	}
	defer resp1.Body.Close()

  

7

**Similarly, developers can use any tool or language they prefer to interact with the microservices and automate tests inside the K8s cluster**

  

8

Once you are done working, you can terminate the telepresence connection to the K8s cluster

  

#Run the below command to disconnect  telepresence 
$ telepresence quit
Telepresence Daemons disconnecting...done

  

  

To clean up the K8s cluster we can use Helm to uninstall the deployment

  

$ helm uninstall app1
release "app1" uninstalled

$ helm uninstall app2
release "app2" uninstalled
