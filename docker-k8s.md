# Deploying the Docker Application and mysql with Volume Support into Kubernetes. (From code to Docker Registries like ACR/ECR and then to EKS/AKS)

### Deploying Python Flask Rest Application to Kubernetes
* In this series we would take a simple python flask Application with the architecture as shown below
![Preview](./Images/dk.png)
* The code of this application is hosted over [Refer Here](https://github.com/DevProjectForDevOps/StudentCoursesRestAPI)

#### Understanding Application Execution
* Before we build the docker image for this application, lets understand the steps for executing this application.
```
git clone https://github.com/DevProjectsForDevOps/StudentCoursesRestAPI.git
cd StudentCoursesRestAPI
pip install -r requirements.txt
python app.py
```
* This application is configured to run on port 8080. so navigate to the `http://<ip address>:8080`. Home page will be launched with swagger ui, where the apis can be executed.

#### Building Docker Image for this application
* Based on the steps mentioned over the above section, we need a source docker image with python3 and pip
* I would be using official python image with tag 3.7-alpine.
* Dockerfile will have the following contents
```
FROM python:3.7-alpine
LABEL author=KHAJA
LABEL blog=directdevops.blog
ARG HOME_DIR='/studentcourses'
ADD . $HOME_DIR
ENV MYSQL_USERNAME='directdevops'
ENV MYSQL_PASSWORD='directdevops'
ENV MYSQL_SERVER='localhost'
ENV MYSQL_SERVER_PORT='3306'
ENV MYSQL_DATABASE='test'
EXPOSE 8080
WORKDIR $HOME_DIR
RUN pip install -r requirements.txt
ENTRYPOINT ["python", "app.py"]
```
* Now build the image using the following command

  `docker build image -t studentcourserestservice:1.0 .`
* Now to test this image we need the mysql container from [Refer Here](https://hub.docker.com/_/mysql) to be created. Lets create the mysql container using the following command

`docker container run -d --name mysql -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=test -e MYSQL_USER=directdevops -e MYSQL_PASSWORD=directdevops mysql:5.6`
* Find the ip address of the mysql container using

  `docker inspect mysql | grep IPAddress`

  and make the note of ip. (In my case it is 172.17.0.2)
* Now create a container with studentcourserestservice image using the following command

  `docker container run -d --name mypythonapp -e MYSQL_SERVER=172.17.0.2 -p 8080:8080 studentcourserestservice:1.0`
* Now navigate to http:<ip address>:8080 and initialize database and select Execute button
![Preview](./Images/dk1.png)
* Now create a sample course by using post request and test it with Get
* Now if every thing is working we are good for pushing this image to our registry.
* I will be pushing this images to AWS Elastic container registry
#### Registry Creation in AWS and pushing image to AWS Elastic Container Registry
* Create Registry in AWS following instructions from [Refer Here](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
* Install AWS-CLI on the machine by following instructions from [Refer Here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and create the iam user [Refer Here](https://sst.dev/chapters/create-an-iam-user.html)
* Execute the following command

  `aws ecr get-login`
* Now execute the docker login command that was returned on the execution of previous command. remove `-e None` from the previous command in case of errors
* Create repository in ECR with name studentcourserestservice
* Create one more repository in ECR with mysql
* Now lets tag our image with latest and version 1.0
```
docker image tag studentcourserestservice:1.0 <awsaccountid>.dkr.ecr.us-west-2.amazonaws.com/studentcourserestservice:1.0 <awsaccountid>.dkr.ecr.us-west-2.amazonaws.com/mysql:5.6

docker image tag mysql:5.6
```
* Push both the images to ecr registry
```
docker push <awsaccountid>.dkr.ecr.us-west-2.amazonaws.com/studentcourserestservice:1.0
docker push <awsaccountid>.dkr.ecr.us-west-2.amazonaws.com/mysql:5.6
```
#### Registry Creation in Azure and pushing the image to Azure Container Registy
* Create Registry in Azure following instructions from [Refer Here](https://learn.microsoft.com/en-in/azure/container-registry/container-registry-get-started-portal?tabs=azure-cli)
* Install Azure cli by following instructions from [Refer Here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* Now from Azure cli enter the following command with the name of registry (i will be using directdevops)
```
az acr login --name <nameofrepository>

az acr login --name directdevops
```
* Now lets tag our image to azure registry name `<acrLoginSever>/<image>:<tag>`

`docker image tag studentcourserestservice:1.0 directdevops.azurecr.io/studentcourserestservice:1.0`
* Push the image the azure registry

`docker image push directdevops.azurecr.io/studentcourserestservice:1.0`
#### Now Create an EKS Cluster and use the Elastic Container Registry
* Create an eks cluster by following instructions from [Refer Here](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)
* In this series i will be using eksctl cli to create the cluster
* Install eksctl by following instructions from [Refer Here](https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html#installing-eksctl)
* create a kubernetes cluster with 2 nodes in the us-west-2 region by entering the following command and wait for approximately 10 minutes
```
eksctl create cluster --name learning --version 1.14 --region us-west-2 \
--nodegroup-name direct-devops --node-type t2.medium --nodes 2 --nodes-min 1 \
--nodes-max 5 --node-ami auto
```
### How about the mysql Storage?
* We will be using mysql:5.7 as the backend of this application and the data stored in mysql needs to be preserved, so we need to use the Volumes.
* Once the kubernetes cluster is created we would need the persistent volume for the mysql container.
* Creating a Storage Class specification with aws-ebs provisioner in a file called aws-storage-class.yml. [Refer Here](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs) for complete info
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  provisioner: kubernetes.io/aws-ebs
  parameters:
    type: gp2
    fsType: ext4
    zones: us-west-2a, us-west-2b, us-west-2c
```
* Now create storage class by kubectl apply command
`kubectl apply -f aws-storage-class.yml`
* Now create a Persistent Volume Claim specification in a file called as aws-pvc.yml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-gp2
  labels:
    app: mysql
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
* Now create a mysql deployment specification in a file called aws-mysql-deploy.yml
```
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
  spec:
    containers:
      - name: mysql
        image: 798279872530.dkr.ecr.us-west-2.amazonaws.com/mysql:5.7
        volumeMounts:
          - mountPath: "/var/lib/mysql"
            persistentVolumeClaim:
              claimName: ebs-gp2
        env:
          - name: MYSQL_DATABASE
            value: 'test'
          - name: MYSQL_USER
            value: 'directdevops'
          - name: MYSQL_PASSWORD
            value: 'directdevops'
          - name: MYSQL_ROOT_PASSWORD
            value: 'password'
        ports:
          - name: dbport
            containerPort: 3306
            protocol: TCP
```
* Now create a specification for service called aws-backend-svc.yml
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
  clusterIP: None
```
* Better thing would be to combine all of the above in one file called as mysql-aws.yml with following content
```
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-gp2
  labels:
    app: mysql
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  minReadySeconds: 10
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: 798279872530.dkr.ecr.us-west-2.amazonaws.com/mysql:5.6
          volumeMounts:
            - mountPath: "/var/lib/mysql"
              name: mysql-vol
          env:
            - name: MYSQL_DATABASE
              value: 'test'
            - name: MYSQL_USER
              value: 'directdevops'
            - name: MYSQL_PASSWORD
              value: 'directdevops'
            - name: MYSQL_ROOT_PASSWORD
              value: 'password'
          ports:
            - name: dbport
              containerPort: 3306
              protocol: TCP
      volumes:
        - name: mysql-vol
          persistentVolumeClaim:
            claimName: ebs-gp2
```
* Now execute the command `kubectl apply -f mysql-aws.yml`
* Now lets create a deployment and service for the python flask application. Remember we need to set the MYSQL_SERVER environment variable and we will be passing the mysql-svc name over here
```
---
apiVersion: v1
kind: Service
metadata:
  name: studentcourse-svc
  labels:
    app: studentscourse
spec:
  ports:
    - port: 8080
  selector:
    app: studentscourse
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentcourse-deploy
  labels:
    app: studentscourse
spec:
  replicas: 5
  selector:
    matchLabels:
      app: studentscourse
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 1
  template:
    metadata:
      labels:
        app: studentscourse
    spec:
      containers:
        - image: 798279872530.dkr.ecr.us-west-2.amazonaws.com/studentcourserestservice
          name: studentscourse
          env:
            - name: MYSQL_SERVER
              value: mysql-svc
          ports:
            - containerPort: 8080
              name: flaskport
```
* Now execute the following commands
```
kubectl apply -f flask-aws.yml
kubectl get svc
```
* In the svc command output you should the load balancer ip.
* Navigate to the application using http:<loadbalancer>:port. In the image shown below, the port exposed on load balancer is 8080
![Preview](./Images/dk2.png)
* Now delete the kubernetes objects created and delete the cluster using eksctl.
* Now from the knowledge achieved from this exercise, try creating the similar deployment into Azure Kubernetes Services
