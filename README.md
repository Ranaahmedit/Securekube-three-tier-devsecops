# Three-Tier Web Application Deployment on AWS EKS using Terraform, AWS EKS, ArgoCD, Prometheus, Grafana, and Jenkins

![Three-Tier](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/73a5368c-d90e-45ab-8978-c0820298b516)


## Introduction
Welcome to the Three-Tier Web Application Deployment project! 🚀 This project aims to provide a comprehensive walkthrough for setting up a robust Three-Tier architecture on AWS using Kubernetes, DevOps best practices, and security measures. It offers hands-on experience in deploying, securing, and monitoring a scalable application environment.

## Project Overview
This project covers the following key aspects:

1. **IAM User Setup:** Creating an IAM user on AWS with necessary permissions.
2. **Infrastructure as Code (IaC):** Using Terraform and AWS CLI to set up Jenkins server.
3. **Jenkins Server Configuration:** Installing and configuring essential tools on Jenkins server.
4. **EKS Cluster Deployment:** Utilizing eksctl commands to create an Amazon EKS cluster.
5. **Load Balancer Configuration:** Configuring AWS Application Load Balancer (ALB) for the EKS cluster.
6. **Amazon ECR Repositories:** Creating private repositories for Docker images on Amazon ECR.
7. **ArgoCD Installation:** Setting up ArgoCD for continuous delivery and GitOps.
8. **Sonarqube Integration:** Integrating Sonarqube for code quality analysis.
9. **Jenkins Pipelines:** Creating pipelines for deploying backend and frontend code to EKS cluster.
10. **Monitoring Setup:** Implementing monitoring using Helm, Prometheus, and Grafana.
11. **ArgoCD Application Deployment:** Using ArgoCD to deploy the Three-Tier application.
12. **Data Persistence:** Implementing persistent volume and claims for database pods.
13. **Conclusion and Monitoring:** Summarizing achievements and monitoring EKS cluster's performance.


## Prerequisites
Before starting the project, ensure you have the following prerequisites:
- An AWS account with necessary permissions.
- Terraform and AWS CLI installed.
- Basic familiarity with Kubernetes, Docker, Jenkins, Argocd, and DevOps principles.


## Setup Steps
For detailed instructions, refer to the following sections:

1. **IAM User Setup**
   - Create an IAM user on AWS with necessary permissions.

2. **Terraform & AWS CLI Installation to deploy our Jenkins Server(EC2) on AWS.**
   - Terraform Installation Script
     
     ```bash
     wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg - dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
     echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
     sudo apt update
     sudo apt install terraform -y
     ```
   - AWSCLI Installation Script
     
     ```bash
     curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
     sudo apt install unzip -y
     unzip awscliv2.zip
     sudo ./aws/install
     ```
   - Configure Terraform
     
     ```bash
     sudo vim /etc/environment
     ```
   - Configure AWS CLI
     
     ```bash
     aws configure
     ```

3. **Deploy the Jenkins Server(EC2) using Terraform**
   (create s3 bucket and dynamodb table manually on AWS Cloud).
   
   - Initialize the backend by running the below command:
     
     ```bash
      terraform init
     ```
     
   -  Now, run the below command to create the infrastructure on AWS Cloud which will take 3 to 4 minutes maximum
     
      ```bash
        terraform apply -var-file=variables.tfvars --auto-approve
      ```

4. **Jenkins Configuration**
   - We have installed some services such as Jenkins, Docker, Sonarqube, Terraform, Kubectl, AWS CLI, and Trivy.
   - Validate installed services
     
     ```bash
     jenkins --version
     docker --version
     docker ps
     terraform --version
     kubectl version
     aws --version
     trivy --version
     eksctl --version
     ```

5. **We will deploy the EKS Cluster using eksctl commands**
   
   - Create EKS cluster
     
     ```bash
       eksctl create cluster --name rana-eks  --region us-east-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
       aws eks update-kubeconfig --region us-east-1 --name rana-eks
     ```
   - Once your cluster is created, you can validate whether your nodes are ready or not by the below command:
     
      ```bash
         kubectl get nodes
      ```

6. **Now, we will configure the Load Balancer on our EKS because our application will have an ingress controller**
   
   - Download the policy for the LoadBalancer prerequisite
     
      ```bash
        curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
      ```
   - Create the IAM policy using the below command:
     
       ```bash
        aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
      ```
   -  Create OIDC Provider
    
      ```bash
       eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=rana-eks --approve
      ```    
   - Create Service Account
    
       ```bash
        eksctl create iamserviceaccount --cluster=rana-eks --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy- 
        arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-east-1
       ```
   - Run the below command to deploy the AWS Load Balancer Controller
      ```bash
        sudo snap install helm --classic
        helm repo add eks https://aws.github.io/eks-charts
        helm repo update eks
        helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws- 
        loadbalancer-controller
       ```    
   - After 2 minutes, run the command below to check whether your pods are running or not.
     
       ```bash
         kubectl get deployment -n kube-system aws-load-balancer-controller
       ```

7. **We need to create Amazon ECR Private Repositories for both Tiers (Frontend & Backend)**

   ![5](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/ff493751-3d70-4679-92c9-d6596aea23ea)
   
   
8. **ArgoCD Installation**
    
    - We will be deploying our application on a three-tier namespace. To do that, we will create a three-tier namespace on EKS
      
    -  As you know, Our two ECR repositories are private. So, when we try to push images to the ECR Repos it will give us the error Imagepullerror.

      To get rid of this error, we will create a secret for our ECR Repo by the below command and then, we will add this secret to the deployment file.

      Note: The Secrets are coming from the .docker/config.json file which is created while login the ECR in the earlier steps
   
    ```bash
         kubectl create secret generic ecr-registry-secret \
         --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
         --type=kubernetes.io/dockerconfigjson --namespace three-tier
         kubectl get secrets -n three-tier
     ```  
       
   - Now, we will install argoCD.
     
   To do that, create a separate namespace for it and apply the argocd configuration for installation.
   
    ```bash
      kubectl create namespace argocd
      kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
      ```
   - Now, expose the argoCD server as LoadBalancer using the below command:
    
       ```bash
        kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
       ```
    
9. **Sonarqube Configuration**
    
    - To do that, copy your Jenkins Server public IP and paste it on your favorite browser with a 9000 port
    - The username and password will be admin
    -  create webhook
        Provide the name of your project and in the URL, provide the Jenkins server public IP with port 8080 add sonarqube-webhook in the suffix, and click on Create.
       http://<jenkins-server-public-ip>:8080/sonarqube-webhook/
   - Now, we have to create a Project for frontend code and frontend code.
     
      ![7](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/94bce182-6fc1-4ab0-ab3b-bb91838660c5)
     
11. **Jenkins Plugins Installation and Configuration**
    - Install the required plugins and configure the plugins to deploy our Three-Tier Application
    - Install the following plugins by going to Dashboard -> Manage Jenkins -> Plugins -> Available Plugins
      
     ```bash
      Docker
      Docker Commons
      Docker Pipeline
      Docker API
      Docker-build-step
      Eclipse Temurin installer
      NodeJS
      OWASP Dependency-Check
      SonarQube Scanner
      AWS Credentials
      Pipeline: AWS Steps
       ```
    - we have to store the credentials for jenkins

       ![4](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/e3247d2d-efb2-4726-8a30-4562b4a6aa8f)


12. **Jenkins Pipelines**
    
    - we are ready to create our Jenkins Pipeline to deploy our Backend Code and frontend code.
      
      ![pipeline-frontend](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/07828b79-67d0-4c4d-ba0a-57c94c832c71)


      ![pipeline-backend](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/2615a41b-ec40-40fc-a4e3-8dba35c7f6fb)

      

13. **We will set up the Monitoring for our EKS Cluster. We can monitor the Cluster Specifications and other necessary things**
    - We will achieve the monitoring using Helm
    - Add the prometheus repo by using the below command:
      
       ```bash
          helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
       ```
       
    - Install the prometheus
       
       ```bash
          helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
          helm install prometheus prometheus-community/prometheus
       ```
       
    -  Now, we need to access our Prometheus and Grafana consoles from outside of the cluster.
    -  Edit the stable-kube-prometheus-sta-prometheus service
     
        ```bash
         kubectl edit svc stable-kube-prometheus-sta-prometheus
       ```

    - Edit the stable-grafana service

       ```bash
         kubectl edit svc stable-grafana
       ```

    - Modification in the 39th line from ClusterType to LoadBalancer

    - Now, access your Prometheus Dashboard

    - Paste the <Prometheus-LB-DNS>:9090 in your favorite browser and you will see like this
    -  Now, access your Grafana Dashboard

  -
    ![promepro](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/cf28be73-7448-47cd-a7d3-75a442cde266)


16. **ArgoCD Application Deployment**
   -  create application for database and backend and forntend and ingress
   -  Once your Ingress application is deployed. It will create an Application Load Balancer

  -  
     ![8](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/0b8c985d-3cf7-41b1-95c5-05155a099455)


   -  You can see all 4 application deployments in the below snippet.

      ![6](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/e122fbc4-ecb0-44f0-a5be-c96eff43e648)

     
  - Now, you can see your Grafana Dashboard to view the EKS data such as pods, namespace, deployments, etc.
    

    ![grafna2](https://github.com/Ranaahmedit/Securekube-three-tier-devsecops/assets/127610751/5c0b08e4-d505-4645-80c3-56c66b522e91)
    

Each step includes detailed instructions and commands for successful setup and deployment. 🛠️
