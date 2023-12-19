## Building Elastic Kubernetes Service (EKS) With Terraform - Deploying and Packaging applications with Helm

Since project 21, you have had some fragmented experience around kubernetes bootstraping and deployment of containerised applications. This project seeks to solidify your skills by focusing more on best practice set up.

- You will use Terraform to create a Kubernetes EKS cluster and dynamically add scalable worker nodes

- You will deploy multiple applications using HELM

- You will experience more kubernetes objects and how to use them with Helm.

- You will improve upon your CI/CD skills with Jenkins

In Project 21, you created a k8s cluster from Ground-Up. That was quite painful, but very necessary to help you master kubernetes. Going forward, you will not have to do that. Even in the real world, you will hardly ever have to do that. given that cloud providers such as AWS have managed services for kubernetes, they have done all the hard work, and with a few API calls to AWS, you can have a production grade cluster ready to go in minutes. Therefore, in this project, you begin by focusing on EKS, and how to get it up and running using Terraform. Before moving on to other things.

#### Building EKS with Terraform
At this point you already have some Terraform experience. So, you have some work to do. But, you can get started with the steps below. If you have terraform code from Project 16, simply update the code and include EKS starting from number 6 below. Otherwise, follow the steps from number 1

#### Note:
- Use Terraform version v1.0.2

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/e0bc22e7-1675-4987-82c9-c540bca44738)

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/d8dc3ebb-dfbf-4391-b386-dd0d1004210a)


- Open up a new directory on your laptop, and name it eks

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/5cf59bb0-be5b-401d-a941-40b05212d6e2)


- Use AWS CLI to create an S3 bucket

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/92081521-2285-4115-8a1a-439a7c9c29a5)


- Create a file - backend.tf Task for you, ensure the backend is configured for remote state in S3

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/0711f81b-9f27-47bf-a7f9-b62283442412)


- Create a file - network.tf and provision Elastic IP for Nat Gateway, VPC, Private and public subnets.


resource "aws_eip" "nat_gw_elastic_ip" {
  vpc = true

  tags = {
    Name            = "${var.cluster_name}-nat-eip"
    iac_environment = var.iac_environment_tag
  }
}
 
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"

  name = "${var.name_prefix}-vpc"
  cidr = var.main_network_block
  azs  = data.aws_availability_zones.available_azs.names

  private_subnets = [
   
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) - 1)
  ]

  public_subnets = [
    
    for zone_id in data.aws_availability_zones.available_azs.zone_ids :
    cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) + var.zone_offset - 1)
  ]

  enable_nat_gateway     = true
  single_nat_gateway     = true
  one_nat_gateway_per_az = false
  enable_dns_hostnames   = true
  reuse_nat_ips          = true
  external_nat_ip_ids    = [aws_eip.nat_gw_elastic_ip.id]

  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    iac_environment                             = var.iac_environment_tag
  }
  
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
    iac_environment                             = var.iac_environment_tag
  }
  
  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
    iac_environment                             = var.iac_environment_tag
  }
}


Note:
The tags added to the subnets is very important. The Kubernetes Cloud Controller Manager (cloud-controller-manager) and AWS Load Balancer Controller (aws-load-balancer-controller) needs to identify the cluster's. To do that, it querries the cluster's subnets by using the tags as a filter.

For public and private subnets that use load balancer resources: each subnet must be tagged


Key: kubernetes.io/cluster/cluster-name
Value: shared



For private subnets that use internal load balancer resources: each subnet must be tagged


Key: kubernetes.io/role/internal-elb
Value: 1



For public subnets that use internal load balancer resources: each subnet must be tagged


Key: kubernetes.io/role/elb
Value: 1


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/2649e7c8-bc3c-4e63-a9e8-4c5c7d9ca247)


- Create a file - variables.tf

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/9f1b0368-bced-4576-bada-3a6267d50f7c)


Create a file - eks.tf and provision EKS cluster (Create the file only if you are not using your existing Terraform code. Otherwise you can simply append it to the main.tf from your existing code) Read more about this module from the official documentation here - Reading it will help you understand more about the rich features of the module.


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/a064b8d0-6c77-46f6-8a7c-98d28bec338a)


- Create a file - worker-nodes.tf - This is used to set the policies for autoscaling. To save as much as 90% of cost we will use Spot Instances - Read more here

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/00994e9e-749a-4e13-9b3a-8e24db9efeec)

- Create a file - locals.tf to create local variables. Terraform does not allow assigning variable to variables. There is good reasons for that to avoid repeating your code unecessarily. So a terraform way to achieve this would be to use locals so that your code can be kept DRY

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/7b628ca1-d678-40e1-9f76-05028ec74460)

- Add more variables to the variables.tf file

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/a2777462-66a5-47bf-8802-08f311061d86)

- Create a file - variables.tfvars to set values for variables.

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/264334cf-f933-4168-9fd7-386c3c117d7f)

- Create file - provider.tf

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/a86ca0ce-82e6-41ff-9d85-ef38e3279252)

- Run terraform init

- Run Terraform plan - Your plan should have an output

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/9d48e494-058b-4259-a800-7201956d2ed5)

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/dd2bb2aa-d5ed-49af-bd1f-ecee34814086)

- Run Terraform apply

This will begin to create cloud resources, and fail at some point with the error

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/9310aeab-9f02-48dd-b98b-62fa1d023561)

That is because for us to connect to the cluster using the kubeconfig, Terraform needs to be able to connect and set the credentials correctly.
To fix this problem

- Append to the file data.tf

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/601a9538-f28b-44d8-ae8e-03ddb31789d2)

- Append to the file provider.tf

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/9e91ec36-07e6-4a67-82db-e50a143de413)

- Run the init and plan again - This time you will see

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/856ed8e8-2279-4ff7-9716-0fb3310a9b8c)


#### Deploy applications with Helm
In Project 22, you experienced the use of manifest files to define and deploy resources like pods, deployments, and services into Kubernetes cluster. Here, you will do the same thing except that it will not be passed through kubectl. In the real world, Helm is one of the most popular tools used to deploy resources into kubernetes. That is because it has a rich set of features that allows deployments to be packaged as a unit. Rather than have multiple YAML files managed individually - which can quickly become messy.
A Helm chart is a definition of the resources that are required to run an application in Kubernetes. Instead of having to think about all of the various deployments/services/volumes/configmaps/ etc that make up your application, you can use a command like

helm install stable/mysql


and Helm will make sure all the required resources are installed. In addition you will be able to tweak helm configuration by setting a single variable to a particular value and more or less resources will be deployed. For example, enabling slave for MySQL so that it can have read only replicas.
Behind the scenes, a helm chart is essentially a bunch of YAML manifests that define all the resources required by the application. Helm takes care of creating the resources in Kubernetes (where they don't exist) and removing old resources.

Lets begin to gradually walk through how to use Helm (Credit - https://andrewlock.net/series/deploying-asp-net-core-applications-to-kubernetes/) @igor please update the texts as much as possible to reduce plagiarism

#### Parameterising YAML manifests using Helm templates

Let's consider that our Tooling app have been Dockerised into an image called tooling-app, and that you wish to deploy with Kubernetes. Without helm, you would create the YAML manifests defining the deployment, service, and ingress, and apply them to your Kubernetes cluster using kubectl apply. Initially, your application is version 1, and so the Docker image is tagged as tooling-app:1.0.0. A simple deployment manifest might look something like the following:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tooling-app-deployment
  labels:
    app: tooling-app
spec:
  replicas: 3
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: tooling-app
  template:
    metadata:
      labels:
        app: tooling-app
    spec:
      containers:
      - name: tooling-app
        image: "tooling-app:1.0.0"
        ports:
        - containerPort: 80


Now lets imagine that the developers develops another version of the toolin app, version 1.1.0. How do you deploy that? Assuming nothing needs to be changed with the service or other kubernetes objects, it may be as simple as copying the deployment manifest and replacing the image defined in the spec section. You would then re-apply this manifest to the cluster, and the deployment would be updated, performing a rolling-update.
The main problem with this is that all of the values specific to the tooling app – the labels and the image names etc – are mixed up with the entire definition of the manifest.
Helm tackles this by splitting the configuration of a chart out from its basic definition. For example, instead of hard coding the name of your app or the specific container image into the manifest, you can provide those when you install the "chart" (More on this later) into the cluster.
For example, a simple templated version of the tooling app deployment might look like the following:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: "{{ template "name" . }}"
spec:
  replicas: 3
  strategy: 
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: "{{ template "name" . }}"
  template:
    metadata:
      labels:
        app: "{{ template "name" . }}"
    spec:
      containers:
      - name: "{{ template "name" . }}"
        image: "{{ .Values.image.name }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80


This example demonstrates a number of features of Helm templates:
The template is based on YAML, with {{ }} mustache syntax defining dynamic sections.
Helm provides various variables that are populated at install time. For example, the {{.Release.Name}} allows you to change the name of the resource at runtime by using the release name. Installing a Helm chart creates a release (this is a Helm concept rather than a Kubernetes concept).
You can define helper methods in external files. The {{template "name"}} call gets a safe name for the app, given the name of the Helm chart (but which can be overridden). By using helper functions, you can reduce the duplication of static values (like tooling-app), and hopefully reduce the risk of typos.
You can manually provide configuration at runtime. The {{.Values.image.name}} value for example is taken from a set of default values, or from values provided when you call helm install. There are many different ways to provide the configuration values needed to install a chart using Helm. Typically, you would use two approaches:
A values.yaml file that is part of the chart itself. This typically provides default values for the configuration, as well as serving as documentation for the various configuration values.
When providing configuration on the command line, you can either supply a file of configuration values using the -f flag. We will see a lot more on this later on.
Now lets setup Helm and begin to use it.

According to the official documentation here, there are different options to installing Helm. But we will build the source code to create the binary.


- Download the tar.gz file from the project's Github release page. Or simply use wget to download version 3.6.3 directly


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/0efe52c0-bbe6-42fd-bbc3-d1ca48b4a451)


- Unpack the tar.gz  file


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/6e5f2571-34c8-450e-b2c4-24a4062451e9)



- cd into the unpacked directory


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/f15b586e-0570-468f-91eb-255828aaa0ca)



- Build the source code using make utility


make build

If you do not have make installed or for any other reason, you cannot install the tool, simply use the official documentation  here for other options.

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/ef21a3e8-0775-45e1-bde0-c6f2950cefce)


Helm binary will be in the bin folder. Simply move it to the bin directory on your system. You cna check other tools to know where that is. fOr example, check where pwd utility is being called from by running which pwd. Assuming the output is /usr/local/bin. You can move the helm binary there.

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/cc7fd8a9-914b-482e-8a03-2f98de9de7c2)


sudo mv bin/helm /usr/local/bin/

Check that Helm is installed

helm version

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/a2da38d0-24d9-47d9-b048-6b465f6eb27f)


#### Deploy Jenkins with Helm
Before we begin to develop our own helm charts, lets make use of publicly available charts to deploy all the tools that we need.
One of the amazing things about helm is the fact that you can deploy applications that are already packaged from a public helm repository directly with very minimal configuration. An example is Jenkins.

Visit Artifact Hub to find packaged applications as Helm Charts
Search for Jenkins

- Add the repository to helm so that you can easily download and deploy


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/ce298566-2f98-441d-8350-b46232e5cd95)


- Update helm repo


![](https://github.com/UzonduEgbombah/project-24/assets/137091610/10f6f018-b8e3-4901-8a3d-b2da430f0deb)


- Install the chart


helm install [RELEASE_NAME] jenkins/jenkins --kubeconfig [kubeconfig file]


You should see an output like this

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/56955753-6d62-4ccc-96fd-439af12e2541)

- Check the Helm deployment

helm ls --kubeconfig [kubeconfig file]

kubectl get pods --kubeconfig [kubeconfig file]

Output:

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/55e917c9-5477-4f86-962b-720d18ad30c8)


- Describe the running pod (review the output and try to understand what you see)

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/f20e4e81-8f8a-4fec-813b-8e6f0360fc8f)


- Check the logs of the running pod

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/46ad4fdf-d7de-4d3f-9764-7586de3b2a9c)


- You will notice an output with an error

This is because the pod has a Sidecar container alongside with the Jenkins container. As you can see fromt he error output, there is a list of containers inside the pod [jenkins config-reload] i.e jenkins and config-reload containers. The job of the config-reload is mainly to help Jenkins to reload its configuration without recreating the pod.
Therefore we need to let kubectl know, which pod we are interested to see its log. Hence, the command will be updated like:

kubectl logs jenkins-0 -c jenkins --kubeconfig [kubeconfig file]

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/089b96fc-a20e-43b6-b7a8-22fd246d5ba1)

Now lets avoid calling the [kubeconfig file] everytime. Kubectl expects to find the default kubeconfig file in the location ~/.kube/config. But what if you already have another cluster using that same file? It doesn't make sense to overwrite it. What you will do is to merge all the kubeconfig files together using a kubectl plugin called [konfig](https://github.com/corneliusweig/konfig) and select whichever one you need to be active.


- Install a package manager for kubectl called krew so that it will enable you to install plugins to extend the functionality of kubectl. Read more about it [Here](https://github.com/kubernetes-sigs/krew)

- Install the [konfig plugin](https://github.com/corneliusweig/konfig)

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/6f5d1062-4270-4042-85cf-58aabef37395)


- Import the kubeconfig into the default kubeconfig file. Ensure to accept the prompt to overide.

sudo kubectl konfig import --save  [kubeconfig file]

- Show all the contexts
Meaning all the clusters configured in your kubeconfig,
If you have more than 1 Kubernetes clusters configured, you will see them all in the output.

kubectl config get-contexts

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/ce789f9b-9f2c-4d65-96a6-f604b579fac7)


- Set the current context to use for all kubectl and helm commands

kubectl config use-context [name of EKS cluster]

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/865f897e-04d1-4e7d-8245-435b184a8e9e)


- Test that it is working without specifying the --kubeconfig flag

kubectl get po

Output:

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/084be1b6-593d-4e45-a7cd-3b14323a0bee)


- Display the current context. This will let you know the context in which you are using to interact with Kubernetes.

kubectl config current-context

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/a69d0bbb-f839-4197-a6ec-c7a03449786b)


Now that we can use kubectl without the --kubeconfig flag, Lets get access to the Jenkins UI. (In later projects we will further configure Jenkins. For now, it is to set up all the tools we need)

There are some commands that was provided on the screen when Jenkins was installed with Helm. See number 5 above. Get the password to the admin user

kubectl exec --namespace default -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo

- Use port forwarding to access Jenkins from the UI

kubectl --namespace default port-forward svc/jenkins 8080:8080

![](https://github.com/UzonduEgbombah/project-24/assets/137091610/97f13f57-d51d-4050-a1b2-30a1475ffa4d)


- Go to the browser localhost:8080 and authenticate with the username and password from number 1 above
   
![](https://github.com/UzonduEgbombah/project-24/assets/137091610/d0a24ac3-b1f1-4d85-a365-5d27c1305353)
















