# Backup EventStreams Using Camel S3 Connector

## Objective
Make a backup of a source Event Streams. The backup should include all topics data (including Schema Registry). Backed up data will be stored in IBM Cloud Object Storage (COS). Consumer lags are not backed up.  


## Pre-requisite

1. Should have a backup Kafka Cluster - preferably in same version as source. 
2. KafkaConnect Cluster not needed.
3. Camel-K Community Operator installed
4. Following information about a IBM COS. Refer [here](./creating_icos.md) on how to create buckets and get details of a ICOS.   
	Access-Key.  
	Secret.  
	Bucket Name.  
	Public Endpoint of IBM COS:   

## Setup Camel-K

### To be done in Both (Source and Target) EventStreams Namespace
1. Install the Camel-K Community Operator from Operator Hub (if not done yet). You can install it in the same namespace as EventStreams.     
![](images/10.jpg).  
Once installed, you should be able to see a list of installed Kamelets. 

	`oc -n <NAMESPACE> get Kamelet`
	
2. In the list displayed, make sure the following 2 Kamelets are listed as we will be using them.   

> aws-s3-sink.  
> asw-s3-source	   

![](images/11.jpg)

### To be done in the Source Namespace

This is the step to backup the contents to Object Storage. 

3. Create a s3-sink KamletBinding. You can use the sample KamletBinding yaml file provided [here](./aws-s3-sink-bind.yaml). Create the KamletBinding.  
`oc -n <NAME-SPACE> apply -f <yaml file>`    
The main parameters that must be checked and changed are:   
bootstrapServers, topic and the connection parameters. For testing purposes, you can use a PLAINTEXT connection.   
S3 connection details: bucketNameOrArn, accessKey, secretKey, overrideEndpoint (MUST be set to true), uriEndpointOverride.      

4. The process will take some time (about 5 minutes) before you see some new pods. If it's successful, you should see at least 2 new pods.  
	> One build pod in completed state.   
	One sink-binding pod in running state.   
	
![](./images/2.png).  

You can also check the status of the KameletBinding.   

	oc -n <NAMESPACE> get kameletbinding.  

![](./images/3.png).  

Next test producing messages to the topic and check the ICOS bucket. 

### To be done in the Target Namespace

This is the step to restore the contents from Object Storage to Kafka Topic.   

1. Create a s3-source KamletBinding. You can use the sample KamletBinding yaml file provided [here](./aws-s3-source-bind.yaml). Create the KamletBinding.  
`oc -n <NAME-SPACE> apply -f <yaml file>`    
The main parameters that must be checked and changed are:   
bootstrapServers, topic and the connection parameters. For testing purposes, you can use a PLAINTEXT connection.   
S3 connection details: bucketNameOrArn, accessKey, secretKey, overrideEndpoint (MUST be set to true), uriEndpointOverride.      


2. The process will take some time (about 5 minutes) before you see some new pods. If it's successful, you should see at least 2 new pods.  
	> One build pod in completed state.   
	One sink-binding pod in running state.   
	
![](images/4.png).  

You can also check the status of the KameletBinding.   

	oc -n <NAMESPACE> get kameletbinding.  

![](./images/5.png).  

You should be able to see that the topic is being created and getting populated with messages from ICOS.   
