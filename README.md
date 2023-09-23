# Project 14: App Deployment on Kubernetes Cluster

This project deploys a WebApp on a kubernetes cluster hosted on AWS.

## Prereqs
- Purchase a domain for kubernetes DNS records
- AWS account
- EC2 instance with linux OS and awscli and kubectl installed

## Steps

1. Setup kops 
   - Create an EC2 instance
    ```
    name: kops
    os: ubuntu 22.04
    keypair: create -> kops-keypair.pem
    security group: 
        name: kopsSg
        rules: allow ssh from myIp
    ```
    - Create s3 bucket:
    ```
     name: vprofile-kops-state124
     region: same working region
    ```
    - Create IAM role and attach it to the kops instance
    ```
     name: kopsAdmin
     policy: AdministratorAccess
    ```
    - SSH into the kops instance generate ssh key
      
        ```
        ssh-keygen
        ```
        install kubectl
        ```
          curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

          chmod +x ./kubectl
          sudo mv kubctl /usr/local/bin/
        ```
        install kops
        ```
           wget https://github.com/kubernetes/kops/releases/download/v1.26.4/kops-linux-amd64
           chmod +x kops-linux-amd64
           sudo mv kops-linux-amd64 /usr/local/bin/kops
        ```
    - create cluster with kops
      ```
      kops create cluster --name=ndzenyuyjones.link --state=s3://<bucket-name> --zones=us-east-2a,us-east-2b --node-count=2 --node-size=t3.small --master-size=t3.medium --dns-zone=ndzenyuyjones.link --node-volume-size=8 --master-volume-size=8
      ```
      This command will create configuration file of a cluster of a master node(t3.medium) and two worker nodes (t3.small) with volume size of the nodes to be 8Gb. this information will be stored in the s3 buck \

      ```
         kops update cluster --name ndzenyuyjones.link --state=s3://<bucket-name> --yes --admin
      ```
      The cluster will be created, we need now to validate the cluster after about 15minutes
      ```
       kops validate cluster --state=s3://vprofile-kops1245 --wait 10m
      ```




      





 