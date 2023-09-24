# Project 14: App Deployment on a Kubernetes Cluster

This project deploys a WebApp on a kubernetes cluster hosted on AWS. In project 6, I containerized the App and stored the images on dockerhub, for this project, those images were pulled and run on a kubernetes cluster. The kubernetes cluster is deployed via a kops VM, it runs a master node and two worker nodes. The different containers run the microservices that make up the full functionality of the project, hence fully decoupled web app. Kubernetes happens to be a very smart container orchestration tool, if a pod fails, it is replaced almost immediately. In my next project, i will be implementing it in a CICD pipeline.

## Architecture
![](https://github.com/Ndzenyuy/Project_14-Webapp-on-Kubernetes/blob/main/images/project14-architecture.jpg)

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

   - Create ebs volume
      ```
         aws ec2 create-volume --availability-zone=us-east-2a --size=3 --volume-type=gp2
      ```
      Copy the volumeID to a safe place for later use

   - Create labels
     Label nodes in us-east-2a: first run
      ```
        kubectl describe node <name of one of the running child nodes> | grep us-east-2
      ```
      If the highligted output reads us-east-2a, then we confirm the node is in us-east-2a, so we label it with "zone=us-east-2a" we run the following command to do it
      ```
      kubectl label nodes <name of the node used in previous command> zone=us-east-2a
      ```
      Label the second node
      ```
      kubectl label nodes <name of the second node > zone=us-east-2b
      ```

2. Create a project in VSCODE
   Create the following files in the project with their contents
   - app-secret.yaml
      ```
      apiVersion: v1
      kind: Secret
      metadata:
      name: app-secret
      type: Opaque
      data:
      db-pass: dnByb2RicGFzcw==
      rmq-pass: Z3Vlc3Q=

      ```
   - db-cip.yaml
      ```
      apiVersion: v1
      kind: Service
      metadata:
      name: vprodb
      spec:
      ports:
      - port: 3306
         targetPort: vprodb-port
         protocol: TCP
      selector:
         app: vprodb
      type: ClusterIP

      ```
   - mc-cip.yaml
      ```
      apiVersion: v1
      kind: Service
      metadata:
      name: vprocache01
      spec:
      selector:
         app: vpromc
      ports:
      - port: 11211
         targetPort: vpromc-port
         protocol: TCP
      type: ClusterIP
      ```
   - mcdep.yaml
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: vpromc
      labels: 
         app: vpromc
      spec:
      selector:
         matchLabels:
            app: vpromc
      replicas: 1
      template:
         metadata:
            labels:
            app: vpromc
         spec:
            containers:
            - name: vpromc
               image: memcached
               ports:
                  - name: vpromc-port
                  containerPort: 11211

      ```
   - rmq-cip.yaml
      ```
      apiVersion: v1
      kind: Service
      metadata:
      name: vpromq01
      spec:
      selector:
         app: vpromq01
      ports:
      - port: 15782
         targetPort: vpromq01-port
         protocol: TCP
      type: ClusterIP
      ```
   - rmq-dep.yaml
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: vpromq01
      labels: 
         app: vpromq01
      spec:
      selector:
         matchLabels:
            app: vpromq01
      replicas: 1
      template:
         metadata:
            labels:
            app: vpromq01
         spec:
            containers:
            - name: vpromq01
            image: rabbitmq        
            ports:
               - name: vpromq01-port
                  containerPort: 15672
            
            env:
               - name: RABBITMQ_DEFAULT_PASS
                  valueFrom:
                  secretKeyRef:
                     name: app-secret
                     key: rmq-pass
               - name: RABBITMQ_DEFAULT_USER
                  value: "guest"

      ```
   - vproappdep.yaml
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: vproapp
      labels: 
         app: vproapp
      spec:
      replicas: 1
      selector:
         matchLabels:
            app: vproapp
      template:
         metadata:
            labels:
            app: vproapp
         spec:
            containers:
            - name: vproapp
            image: ndzenyuy/vprofileapp:latest
            ports:
            - name: vproapp-port
               containerPort: 8080
            initContainers:
            - name: init-mydb
            image: busybox
            command: ['sh', '-c', 'until nslookup vprodb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done;']
            - name: init-memcache
            image: busybox
            command: ['sh', '-c', 'until nslookup vprocache01.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done;']
      ```
   - vproapp-service.yaml
      ```
      apiVersion: v1
      kind: Service
      metadata:
      name: vproapp-service
      spec:
      selector:
         app: vproapp
      ports:
      - port: 80
         targetPort: vproapp-port
         protocol: TCP
      type: LoadBalancer
      ```
   - vproappingress.yaml
      ```
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
      name: vproapp-ingress
      annotations:
         nginx.ingress.kubernetes.io/rewrite-target: /
      spec:
      ingressClassName: nginx
      rules:
      - http:
            paths:
            - path: /
            pathType: Prefix
            backend:
               service:
                  name: vproapp-service
                  port:
                  number: 80
      ```
   - vprodbdep.yaml
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: vprodb
      labels: 
         app: vprodb
      spec:
      selector:
         matchLabels:
            app: vprodb
      replicas: 1
      template:
         metadata:
            labels:
            app: vprodb
         spec:
            containers:
            - name: vprodb
            image: ndzenyuy/vprofiledb:latest
            volumeMounts:
               - mountPath: /var/lib/mysql
                  name: vpro-db-data
            ports:
               - name: vprodd-port
                  containerPort: 3306
            env:
               - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                  secretKeyRef:
                     name: app-secret
                     key: db-pass
            nodeSelector: 
            zone: us-east-2a
            volumes:
            - name: vpro-db-data
               awsElasticBlockStore: 
                  volumeID: <the volume id that was stored earlier>
                  fsType: ext4

            initContainers:
            - name: busybox
               image: busybox:latest
               args: ["rm", "-rf", "/var/lib/mysql/lost+found"]
               volumeMounts:
                  - name: vpro-db-data
                  mountPath: /var/lib/mysql
      ```

      Commit the project into a github repository.

   Now edit the tags of the created ebs volume,
   ```
   Name: KubernetesCluster
   value: <cluster-name> eg kubernetes.ndzenyuyjones.link
   ```
   ![](https://github.com/Ndzenyuy/Project_14-Webapp-on-Kubernetes/blob/main/images/edit%20volume%20tag.png)
   
   3. SSH into kops instance and clone the source code
   ```
   git clone <project_url>
   cd project folder

   ```

   4. Launch the webapp
   ```
   kubectl apply -f .
   ```
   5. Back to AWS console, onto elastic load balancer, copy the url of the ELB that has been created and search it on the browser
   ![](https://github.com/Ndzenyuy/Project_14-Webapp-on-Kubernetes/blob/main/images/elastic%20load%20balancer%20created.png)

   ![](https://github.com/Ndzenyuy/Project_14-Webapp-on-Kubernetes/blob/main/images/login%20page.png)

   








      





 
