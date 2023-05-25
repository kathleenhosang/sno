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
Valid AWS Access Key ID |
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
AWS API Key ID > AKIAWKXUCZ55STYXXX

AWS Secret Access Key
Provide your AWS Secret Access Key (if you have not set the AWS_SECRET_ACCESS_KEY
environment variable) which will be used to provision an AWS instance.

AWS Secret Access Key > HiIoMnhB13tKthkiBlXvpJM9g/znKKlCgJoyxxxx
Re-use saved AWS Secret Access Key Starting 'HiIoMnhB13tKthkiBlXvpJM9g/znKKlCgJoyxxxx'? [Y/n] 

AWS Cluster Configuration
Cluster Name > sno
AWS Region > us-east-2
AWS Base Domain > buyermas4aws.com
Do you want single node openshift  [Y/n] 

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


## Install MAS and MAS Manage

