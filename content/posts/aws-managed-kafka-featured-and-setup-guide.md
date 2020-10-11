---
title: "AWS Managed Kafka. Features And Setup Guide"
date: 2020-10-10
draft: false
---

## Amazon Managed Streaming for Apache Kafka (Amazon MSK)

Amazon MSK is a fully managed service that makes it easy for you to build and run applications that use Apache Kafka to process streaming data. Managing Apache Kafka clusters is complex and time consuming. Amazon MSK makes it easy for you to build and run production applications on Apache Kafka without needing Apache Kafka infrastructure management expertise so you spend less time managing infrastructure and more time building applications.

### Overview

#### Fully compatible
Support for native Apache Kafka APIs and tools
Amazon MSK supports native Apache Kafka APIs and existing open-source tools built against those APIs. This enables existing Apache Kafka applications to work with Amazon MSK clusters without changes to application code. You continue to use Apache Kafka’s APIs and the open-source ecosystem to populate data lakes, stream changes to and from databases, and power machine learning and analytics applications.

#### Fully managed
With a few clicks in the Amazon MSK console, you can create a fully managed Apache Kafka cluster that follows Apache Kafka’s deployment best practices, or you can create your own cluster using your own custom configuration. Once you create your desired configuration, Amazon MSK automatically provisions, configures, and manages the operations of your Apache Kafka cluster and Apache ZooKeeper nodes.

#### Fully Managed Apache Kafka Upgrades
You can upgrade Apache Kafka versions on Amazon MSK clusters in just a few clicks, allowing you to take advantage of features and bug fixes present in new Apache Kafka versions. Amazon MSK provides a fully-managed upgrade experience using a rolling update of Apache Kafka brokers to enable in-place upgrades for customers following high-availability best practices.

#### Highly available
Automatic recovery and patching
Amazon MSK continuously monitors the health of your clusters and replaces unhealthy brokers without downtime for your applications. Amazon MSK manages the availability of your Apache ZooKeeper nodes so you will not need to start, stop, or directly access the nodes yourself. Amazon MSK also deploys software patches as needed to keep your cluster up to date and running smoothly.

#### Data replication
Amazon MSK uses multi-AZ replication for high-availability. Data replication is included at no additional cost.  

#### Private connectivity
Your Apache Kafka clusters run in an Amazon VPC managed by Amazon MSK. Your clusters are available to your own Amazon VPCs, subnets, and security groups based on the configuration you specify. You have complete control of your network configuration, and IP addresses from your VPCs are attached to your Amazon MSK resources through elastic network interfaces (ENIs).

#### Encryption and security
Amazon MSK encrypts your data at rest without special configuration or third-party tools. All data can be encrypted at rest using AWS Key Management Service (KMS) Customer Master Key (CMK) by default, or your own CMK.

Amazon MSK also encrypts data in-transit via TLS between brokers and between clients and brokers on your cluster. Amazon MSK also supports TLS based certificate authentication, SASL/SCRAM authentication secured by AWS Secrets Manager, and Apache Kafka access control lists (ACLs) to authenticate and authorize producers and consumers within your cluster.

#### Broker scaling
You can start with a few brokers within an Amazon MSK cluster. Then, using the AWS management console or AWS CLI, you can scale up to 100’s of brokers per cluster. Soft limit is 15 brokers per cluster or more than 30 brokers per account.

#### Storage scaling
You can seamlessly scale up the amount of storage provisioned per broker. You can create an auto scaling policy to automatically expand your storage to meet your streaming requirements.

#### Low cost
Amazon MSK lets you get started for less than $2.50 per day. 

#### Deeply integrated
Amazon MSK makes it easier for AWS customers to build end-to-end solutions by providing native AWS integrations out-of-the-box. You can run fully managed Apache Flink applications on data within Amazon MSK, encrypt data at rest using AWS KMS, authenticate clients to Amazon MSK using AWS Certificate Manager Private CAs or client credentials secured by AWS Secrets Manager, deploy Amazon MSK using code with AWS CloudFormation, privately connect clients within an Amazon VPC to Amazon MSK, and leverage AWS Identity and Access Management (IAM) for fine-grained service-level API control.

#### Configurable
Amazon MSK deploys a best practice cluster configuration for Apache Kafka by default, and gives customers the ability to tune more than 30 different cluster configurations while supporting all dynamic and topic-level configurations. 

### Amazon MSK pricing
With Amazon MSK, you pay only for what you use. There are no minimum fees or upfront commitments. You pay for the time your broker instances run, the storage you use monthly, and standard data transfer fees for data in and out of your cluster. You do not pay for Apache ZooKeeper nodes that Amazon MSK provisions for you, or data transfer that occurs between brokers and nodes within clusters.

#### Broker instance pricing
You pay for Apache Kafka broker instance usage on an hourly basis (billed at one second resolution), with varying fees depending on the size of the Apache Kafka broker instance and active brokers in your Amazon MSK clusters.

#### Broker storage pricing

With Amazon MSK, you pay for the amount of storage you provision in your cluster. Calculated by `hours * brokers number` as "GB-Months".

#### Data transfer fees
- No data transfer fees between brokers or Zookeper nodes
- Standard AWS data transfer charges for data transferred in and out of Amazon MSK clusters

## Amazon MSK Setup Guide

### Step 1: Create a VPC for Your MSK Cluster

Create CloudFormation Stack form [vpc.cfn.yml](#code-refference) file below to create AWS VPC.  

This template deploys a VPC, with a pair of public and private subnets spread 
across two Availability Zones. It deploys an internet gateway, with a default
route on the public subnets. It deploys a pair of NAT gateways (one in each AZ),
and default routes for them in the private subnets.

### Step 2: Create a MSK Cluster

Create CloudFormation Stack from [msk.cfn.yml](#code-refference) file below to create AWS Managed Kafka.

Also you can go through following optional steps to setup Kafka:
1. Rewrite default configuration in (MSK Configuration Properties)[https://docs.aws.amazon.com/msk/latest/developerguide/msk-default-configuration.html]: 
```yml
auto.create.topics.enable=true
delete.topic.enable=true
log.retention.hours=8
```
2. Enable Encryption within the cluster. Note that this can (impact the performance)[https://docs.aws.amazon.com/msk/latest/developerguide/msk-encryption.html] of the cluster in production.

> Tip: You **cannot enable encryption on an already created cluster**, nor can you turn it off on a cluster configured with encryption, you can only turn on encryption in-transit during cluster creation. Note that this can impact the performance of the cluster in production. If you don’t need this level of encryption consider leaving it off.

### Step 3: Run Workload

Create CloudFormation Stack from [app.cfn.yml](#code-refference) file below to create AWS Managed Kafka.

> 🔥 Don't forget to enable CAPABILITIES_IAM for CloudFormation script.

### Code Refference

You can find detailed code in [GitHub](https://github.com/srcmaxim/aws-msk-setup)

<script src="https://gist.github.com/srcmaxim/32300b5e3acf071305372c07e8fecc76.js"></script>

## Resources 

1. [AWS MSK Docs](https://docs.aws.amazon.com/msk/latest/developerguide)
2. [AWS MSK WORKSHOP](https://amazonmsk-labs.workshop.aws/en/clustercreation.html)
3. [CLICKSTREAM LAB](https://amazonmsk-labs.workshop.aws/en/mskkdaflinklab.html)
  End to end lab for a clickstream use case using Amazon MSK for stream storage and Amazon KDA for Java Applications with the Apache Flink engine for stream processing.
4. [AMAZON MSK LAMBDA INTEGRATION](https://amazonmsk-labs.workshop.aws/en/msklambda.html)
  Explains how integration between AWS Lambda and Amazon MSK works. How to propagate or backup messages from a topic in Amazon MSK to Amazon S3 using Kinesis Data Firehose.
