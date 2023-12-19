Building Elastic Kubernetes Service (EKS) With Terraform - Deploying and Packaging applications with Helm
Since project 21, you have had some fragmented experience around kubernetes bootstraping and deployment of containerised applications. This project seeks to solidify your skills by focusing more on best practice set up.

You will use Terraform to create a Kubernetes EKS cluster and dynamically add scalable worker nodes
You will deploy multiple applications using HELM
You will experience more kubernetes objects and how to use them with Helm.
You will improve upon your CI/CD skills with Jenkins

In Project 21, you created a k8s cluster from Ground-Up. That was quite painful, but very necessary to help you master kubernetes. Going forward, you will not have to do that. Even in the real world, you will hardly ever have to do that. given that cloud providers such as AWS have managed services for kubernetes, they have done all the hard work, and with a few API calls to AWS, you can have a production grade cluster ready to go in minutes. Therefore, in this project, you begin by focusing on EKS, and how to get it up and running using Terraform. Before moving on to other things.

Building EKS with Terraform
At this point you already have some Terraform experience. So, you have some work to do. But, you can get started with the steps below. If you have terraform code from Project 16, simply update the code and include EKS starting from number 6 below. Otherwise, follow the steps from number 1
