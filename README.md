# MAS on SNO
Install Single Node OpenShift + MAS Manage Install on AWS

## Architecture Overview

<img width="785" alt="Screenshot 2023-05-25 at 2 28 47 PM" src="https://github.com/kathleenhosang/sno/assets/40863347/ff048dc6-41bc-466c-8271-472409d54bcf">


## Prerequisities
Requirement | Details |
--------|-------|
OpenShift | 4.10+ |
vCPU | 16Cores |
RAM | 64Gb |
IBM entitlement Key | Log in to the IBM Container Library with a user ID that has software download rights for your company’s Passport Advantage entitlement to get the entitlement key. |
Openshift pull secret file (pull-secret) | It can be downloaded from [here] (https://access.redhat.com/management). You need a valid redhat account for downloading.
MAS license file (license.dat) | Access IBM License Key Center to the Get Keys menu select IBM AppPoint Suites. Select IBM MAXIMO APPLICATION SUITE AppPOINT LIC. |
Docker/Podman |
Valid AWS Access Key ID | This is the access key associated with the user ID you are using to install. |
AWS Secret Access key | If you don't have it, ask your aws account admin to create one in IAM service |
AWS Domain or subdomain | If you don't have one, ask your aws account admin to register one through AWS Route53 |


## Install Single Node OpenShift using mas cli

Documentation: https://ibm-mas-manage.github.io/sno/

_Perform the folloinwg steps from a bastion node (or local machine)._

1. Set up IBM MAS DevOps ansible collection docker container

```sh
mkdir ~/sno
cd ~/sno
docker pull quay.io/ibmmas/cli
docker run -dit --name sno quay.io/ibmmas/cli:latest bash
```

2. Log into the docker container; create a folder for mas configuration; then exit the container.

```sh
docker exec -it sno bash
mkdir masconfig
exit
```

3. Copy pull-secret and mas license file into the docker container

```sh
docker cp pull-secret sno:/mascli/masconfig/pull-secret
docker cp license.dat sno:/mascli/masconfig/license.dat
```

4. Log into docker container

```sh
docker exec -it sno bash

Available commands:
  - mas install to launch a MAS install pipeline
  - mas provision-fyre to provision an OCP cluster on IBM DevIT Fyre (internal)
  - mas provision-roks to provision an OCP cluster on IBMCloud Red Hat OpenShift Service (ROKS)
  - mas provision-aws to provision an OCP cluster on AWS
  - mas provision-rosa to provision an OCP cluster on AWS Red Hat OpenShift Service (ROSA)
  - mas setup-registry to setup a private container registry on an OCP cluster
  - mas mirror-images to mirror container images required by mas to a private registry
  - mas configure-ocp-for-mirror to configure a cluster to use a private registry as a mirror
```

5. Run the command to provision SNO AWS Cluster. It will automatically detect the single enode.
* Enter your AWS credentials:
* AWS API Key ID
* AWS Secret Access Key
* AWS Secret Access Key
* Cluster Name
* AWS Region
* AWS Base Domain

```sh
mas provision-aws

IBM Maximo Application Suite AWS Cluster Provisioner
Powered by https://github.com/ibm-mas/ansible-devops/


AWS Access Key ID
Provide your AWS API Key ID (if you have not set the AWS_ACCESS_KEY_ID
environment variable) which will be used to provision an AWS instance.
AWS API Key ID > XXXXXXXXXXXXXXXXX

AWS Secret Access Key
Provide your AWS Secret Access Key (if you have not set the AWS_SECRET_ACCESS_KEY
environment variable) which will be used to provision an AWS instance.

AWS Secret Access Key > XXXXXXXXXXXXXXXXX
Re-use saved AWS Secret Access Key Starting 'XXXXXXXXXXXXXXX'? [Y/n] 

AWS Cluster Configuration
Cluster Name > sno
AWS Region > us-east-2
AWS Base Domain > dns.com
Do you want single node openshift  [Y/n] y

OCP Version:
  1. 4.10 EUS 
Select Version > 1

Proceed with these settings [y/N] y
```

You see the following message for your cluster after it is provisioned.

```sh
AWS cluster is ready to use
Connected to OCP cluster: https://console-openshift-console.apps.sno.dns.com
```

## Set Up Amazon EFS

Documentation: https://docs.openshift.com/container-platform/4.10/storage/container_storage_interface/persistent-storage-csi-aws-efs.html

1. Install the "AWS EFS CSI Driver Operator" from the OpenShift Operator Hub.

<img width="1681" alt="Screenshot 2023-05-26 at 1 33 34 PM" src="https://github.com/kathleenhosang/sno/assets/40863347/2f56cfc4-d008-4db4-b79d-8785ecbe8c58">

Use the default settings:

<img width="1521" alt="Screenshot 2023-05-26 at 1 37 58 PM" src="https://github.com/kathleenhosang/sno/assets/40863347/c9a4e102-b257-484f-99b3-939b60e9fe4b">

2. Install the AWS EFS CSI Driver:

Click administration → CustomResourceDefinitions → ClusterCSIDriver.

On the Instances tab, click Create ClusterCSIDriver.

Use the following YAML file:

```sh
apiVersion: operator.openshift.io/v1
kind: ClusterCSIDriver
metadata:
    name: efs.csi.aws.com
spec:
  managementState: Managed
```

Wait for the following Conditions to change to a "true" status:
* AWSEFSDriverCredentialsRequestControllerAvailable
* AWSEFSDriverNodeServiceControllerAvailable
* AWSEFSDriverControllerServiceControllerAvailable

3. Configure File Share on AWS for the Storage Class

On the AWS console, open https://console.aws.amazon.com/efs. Click Create file system:

Enter a name for the file system.

For Virtual Private Cloud (VPC), select your OpenShift Container Platform’s' virtual private cloud (VPC).

Accept default settings for all other selections.

Wait for the volume and mount targets to finish being fully created:

Go to https://console.aws.amazon.com/efs#/file-systems.

Click your volume, and on the Network tab wait for all mount targets to become available (~1-2 minutes).

On the Network tab, copy the Security Group ID (you will need this in the next step).

Go to https://console.aws.amazon.com/ec2/v2/home#SecurityGroups, and find the Security Group used by the EFS volume.

On the Inbound rules tab, click Edit inbound rules, and then add a new rule with the following settings to allow OpenShift Container Platform nodes to access EFS volumes:

Type: NFS
Protocol: TCP
Port range: 2049
Source: Custom/IP address range of your nodes (for example: “10.0.0.0/16”)

This step allows OpenShift Container Platform to use NFS ports from the cluster.

4. Create the storage class

Create a apply the StorageClass object yaml:
Replace the fileSystemID with the ID of the newly created AWS file system.

```sh
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap 
  fileSystemId: fs-a5324911 
  directoryPerms: "700" 
  gidRangeStart: "1000" 
  gidRangeEnd: "2000" 
  basePath: "/dynamic_provisioning" 
```

5. Enable dynamic provisioning 

Create a PVC (or StatefulSet or Template) as usual, referring to the StorageClass created above:

```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
spec:
  storageClassName: efs-sc
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```
## Install MAS and MAS Manage

1. Log into the cluster

2. mas install (use EFS storage class to install)






