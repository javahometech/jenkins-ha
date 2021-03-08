# Terraform and Packer templates for Jenkins ASG cluster

Packer and Terraform code to deploy a Jenkins cluster with high-availability.  This repository includes:

* A Packer template and provisioning code to create an AMI suitable for use as Jenkins master or build slave.
* Terraform resources creates an autoscaling cluster with:
  * Single-instance Jenkins master in autoscaling group, with health check and instance replacement;
  * Scalable builder pool autoscaling group, capable of launching on-demand or Spot EC2 builders (under control of EC2-Fleet plugin);
  * Initial configuration and plugins automatically provisioned with JCasC
  * AWS and builder SSH credentials auto-configured via init.groovy

[TOC]

## Build an AMI for the cluster

A Packer template and post-provisioning shell code is provided in the *packer* subdirectory.  A helper script (*buildAMI.sh*) is provided as a guideline for using the Packer template (note that some minor editing of this script may be needed, for instance to point to your packer executable).

Since this manipulates AWS resources, you will need to run the Packer execution with established and suitable AWS session credentials.

After build, the packer job will print the AMI ID of the new image.  You'll want that for cluster deployment parameters (below).

## Deploy a cluster

The repository contains necessary Terraform files for deploying or updating a Jenkins cluster.  A helper script (*deploy.sh*) is provided as a example for building the deployment.  (note that some minor editing of this script may be needed, for instance to point to your terraform executable).

Since this manipulates AWS resources, you will need to run the Terraform execution with established and suitable AWS session credentials.

The Terraform code presumes a JSON-formatted parameter file to collect all needed parameters for the deployment.

## Parameter File

The Terraform code presumes a JSON-formatted parameter file to collect all needed parameters for the deployment.

```json
{
  "jenkins_ami" : "ami-066c04d4324d02a13",
  "jenkins_name" : "jenkins-prod",
  "jenkins_motd" : "Jenkins HA master pre-production\\nThis instance is configured by Terraform and JCasC",
  "jenkins_version" : "2.190.3",
  "vpc_id" : "vpc-3c64ac5a",
  "region" : "us-west-2",
  "master_subnets" : [ "subnet-f3f51095", "subnet-7776493e" ],
  "builder_subnets" : [ "subnet-f3f51095", "subnet-7776493e" ],
  "master_instance_role" : "HPE_Jenkins",
  "master_rootdisk_gb" : "40",
  "builder_rootdisk_gb" : "40",
  "builder_pool_mininstances" : "0",
  "builder_pool_maxinstances" : "4",
  "builder_pool_odcount" : "1",
  "domain_fqdn" : "alstestjenkins.saas.microfocus.com",
  "ssl_certificate_arn" : "arn:aws:iam::986148801171:server-certificate/saasmicrofocuscom_2018",
  "master_keypair" : "sparksa"
}
```

| Parameter                 | Type           | Description                                                  |
| ------------------------- | -------------- | ------------------------------------------------------------ |
| jenkins_name              | String         | A short string identifying the cluster.  EC2 instance names, ASGs, and ELBs will be prefixed with this. |
| jenkins_ami               | String         | EC2 AMI of Jenkins AMI.  Will be used in launch templates for the cluster master/builders. |
| jenkins_motd              | String         | A string to display on the Jenkins master.  Note you can use multiple lines - separate each line with the newline sequence (\n). |
| jenkins_version           | String         | Pin Jenkins version to this version (e.g. "2.190.3").        |
| vpc_id                    | String         | AWS VPC ID in which cluster is launched.  Region and subnet IDs must be consistent with this VPC. |
| region                    | String         | Name of AWS region where cluster is deployed.                |
| master_subnets            | List of String | One or more subnet IDs (enclosed by [], separated by comma) where master instances can be launched. |
| builder_subnets           | List of String | One or more subnet IDs (enclosed by [], separated by comma) where builder instances can be launched. |
| aws_role                  | String         | Name of IAM role to add as "aws" credential in the Jenkins master instance. |
| aws_instance_role         | String         | Name of IAM instance role to be assigned to Jenkins master/builders. |
| master_rootdisk_gb        | String         | Size (in GB) of the master instance root volume.             |
| builder_rootdisk_gb       | String         | Size (in GB) of each builder instance root volume.           |
| builder_pool_mininstances | String         | For builder pool, minimum number of build slaves to run.     |
| builder_pool_maxinstances | String         | For builder pool, maximum number of build slaves to run.     |
| builder_pool_odcount      | String         | For builder pool, number of build slaves to run as on-demand instances.  As builders start, the first N number are started as on-demand; any additional instances more than this are launched as Spot instances. |
| ssl_certificate_arn       | String         | ARN of IAM server certificate to deploy on the cluster Elastic Load Balancer. |
| domain_fqdn               | String         | DNS fully-qualified domain name for the cluster.             |
| master_keypair            | String         | Name of EC2 keypair to assign to launched master or builder instances. |

## Cluster Maintenance

Because of the automated deployment and update, and because the Jenkins master is managed by Autoscaling, some maintenance must be performed by deploying a new AMI, rather than hacking or patching the running instance.  Some maintenance, such as applying OS patches, would be lost if the master was replaced by Autoscaling.

NOTE: when performing a cluster redeployment (with or without a new AMI ID), please follow the below procedure to ensure proper operation after deployment:

* If any change will require the Jenkins master to be halted for more than a couple of minutes, consider disabling scaling activities on the master autoscaling group.  This is to ensure AS does not terminate an instance from underneath you.
* If possible, wait for jobs to finish building, and build slaves to exit.
* Perform the deployment.
* Terminate the current master instance and allow autoscaling to launch a new master.  This is to ensure that the new SSH key credentials are initialized before launching new builders.

### Maintenance that MUST be performed by an AMI rebuild and cluster redeployment

- OS patches (to ensure that Autoscaling instance replacement launches a patched instance);
- Jenkins version upgrade;
- Modification of the tools included in the AMI for builds;
- LDAP authentication changes

### Maintenance that must be performed by a cluster redeployment

- Changes in the Autoscaling parameters (e.g., min and max limits for EC2 build fleet);
- Change in Jenkins message-of-the-day;
- Changes in AWS role names (only role names - role policies may be changed in AWS without redeployment)

### Maintenance that may be applied to the current master without a redeployment

- Plugin updates, or new plugin installation
