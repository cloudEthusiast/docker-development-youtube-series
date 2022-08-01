# Jenkins on Amazon Kubernetes 

## Create a cluster

Follow my Introduction to Amazon EKS for beginners guide, to create a cluster <br/>
Video [here](https://youtu.be/QThadS3Soig)

## Setup our Cloud Storage 

```
# deploy EFS storage driver
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# get VPC ID
aws eks describe-cluster --name eks-mendix \
--query "cluster.resourcesVpcConfig.vpcId" --output text #vpc-059abdda977cacdb0

# Get CIDR range
aws ec2 describe-vpcs --vpc-ids vpc-059abdda977cacdb0  \
 --query "Vpcs[].CidrBlock" --output text #10.0.0.0/16

# security for our instances to access file storage
aws ec2 create-security-group --description efs-test-sg --group-name efs-sg --vpc-id vpc-059abdda977cacdb0 #GroupId: sg-0255594e5ead6a9ac

aws ec2 authorize-security-group-ingress --group-id  sg-0255594e5ead6a9ac \
  --protocol tcp --port 2049 --cidr 10.0.0.0/16

  #Values
  SecurityGroupRules:
- CidrIpv4: 10.0.0.0/16
  FromPort: 2049
  GroupId: sg-0255594e5ead6a9ac
  GroupOwnerId: '469176205295'
  IpProtocol: tcp
  IsEgress: false
  SecurityGroupRuleId: sgr-042b55d66dd2dfcbb
  ToPort: 2049

  5dc9  --subnet-id subnet-0de4c63574b19d26d --security-group sg-0255594e5ead6a9ac
AvailabilityZoneId: euw1-az2
AvailabilityZoneName: eu-west-1a
FileSystemId: fs-06d2496c65bf75dc9
IpAddress: 10.0.4.40
LifeCycleState: creating
MountTargetId: fsmt-084671354067ea1d2
NetworkInterfaceId: eni-0e037afea97b7179f
OwnerId: '469176205295'
SubnetId: subnet-0de4c63574b19d26d
VpcId: vpc-059abdda977cacdb0



# create storage
aws efs create-file-system --creation-token eks-efs

# VAlues
CreationTime: '2022-07-29T23:52:46+01:00'
CreationToken: eks-efs2
Encrypted: false
FileSystemArn: arn:aws:elasticfilesystem:eu-west-1:469176205295:file-system/fs-06d2496c65bf75dc9
FileSystemId: fs-06d2496c65bf75dc9
LifeCycleState: creating
NumberOfMountTargets: 0
OwnerId: '469176205295'
PerformanceMode: generalPurpose
SizeInBytes:
  Value: 0
  ValueInIA: 0
  ValueInStandard: 0
Tags: []
ThroughputMode: bursting


# create mount point 
aws efs create-mount-target --file-system-id FileSystemId --subnet-id SubnetID --security-group GroupID

#Values
vailabilityZoneId: euw1-az3
AvailabilityZoneName: eu-west-1b
FileSystemId: fs-06d2496c65bf75dc9
IpAddress: 10.0.7.30
LifeCycleState: creating
MountTargetId: fsmt-0e902eb4f565426c0
NetworkInterfaceId: eni-0889685eb80b81133
OwnerId: '469176205295'
SubnetId: subnet-0c02abffd2083eddf
VpcId: vpc-059abdda977cacdb0


# grab our volume handle to update our PV YAML
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
```

More details about EKS storage [here](https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/)

### Setup a namespace
```
kubectl create ns jenkins
```

### Setup our storage for Jenkins

```
kubectl get storageclass

# create volume
kubectl apply -f ./jenkins/amazon-eks/jenkins.pv.yaml 
kubectl get pv

# create volume claim
kubectl apply -n jenkins -f ./jenkins/amazon-eks/jenkins.pvc.yaml
kubectl -n jenkins get pvc
```

### Deploy Jenkins

```
# rbac
kubectl apply -n jenkins -f ./jenkins/jenkins.rbac.yaml 

kubectl apply -n jenkins -f ./jenkins/jenkins.deployment.yaml

kubectl -n jenkins get pods

```

### Expose a service for agents

```

kubectl apply -n jenkins -f ./jenkins/jenkins.service.yaml 

```

## Jenkins Initial Setup

```
kubectl -n jenkins exec -it jenkins-6c88cc978b-gd6c8  -- bash

cat /var/jenkins_home/secrets/initialAdminPassword #a27b4ff029ea481381c2195eb7f93912
kubectl port-forward -n jenkins jenkins-b4d464798-lszgk 8080  #jenkins-b4d464798-lszgk 


USERNAME: medix-test
PSSD: Nigeria@#1506, Nigeria@1506


# setup user and recommended basic plugins
# let it continue while we move on!

```

## SSH to our node to get Docker user info

```
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
ssh -i ~/.ssh/id_rsa ec2-user@ec2-13-239-41-67.ap-southeast-2.compute.amazonaws.com
id -u docker #1001
cat /etc/group   #docker:x:1950:ec2-user
# Get user ID for docker
# Get group ID for docker
```

#connnect to ec2-with CLI
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html



## Docker Jenkins Agent

Docker file is [here](../dockerfiles/dockerfile) <br/>

```
# you can build it

cd ./jenkins/dockerfiles/
docker build . -t aimvector/jenkins-slave

```

## Continue Jenkins setup


Install Kubernetes Plugin <br/>
Configure Plugin: Values I used are [here](../readme.md) <br/>

Install Kubernetes Plugin <br/>

## Try a pipeline
 
```
pipeline {
    agent { 
        kubernetes{
            label 'jenkins-slave'
        }
        
    }
    environment{
        DOCKER_USERNAME = credentials('DOCKER_USERNAME')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
    }
    stages {
        stage('docker login') {
            steps{
                sh(script: """
                    docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
                """, returnStdout: true) 
            }
        }

        stage('git clone') {
            steps{
                sh(script: """
                    git clone https://github.com/marcel-dempers/docker-development-youtube-series.git
                """, returnStdout: true) 
            }
        }

        stage('docker build') {
            steps{
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/docker-development-youtube-series/python
                docker build . --network host -t aimvector/python:${BUILD_NUMBER}
                '''
            }
        }

        stage('docker push') {
            steps{
                sh(script: """
                    docker push aimvector/python:${BUILD_NUMBER}
                """)
            }
        }

        stage('deploy') {
            steps{
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/docker-development-youtube-series/
                #get kubectl for this demo
                curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                chmod +x ./kubectl
                ./kubectl apply -f ./kubernetes/configmaps/configmap.yaml
                ./kubectl apply -f ./kubernetes/secrets/secret.yaml
                cat ./kubernetes/deployments/deployment.yaml | sed s/1.0.0/${BUILD_NUMBER}/g | ./kubectl apply -f -
                ./kubectl apply -f ./kubernetes/services/service.yaml
                '''
        }
    }
}
}
```


