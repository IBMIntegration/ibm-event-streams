# Backup EventStreams Using By Copying Logs Folder

## Objective
Make a backup of a source Event Streams. The backup should include all topics data (including Schema Registry). Backed up data will be stored in external storage. Consumer lags are not backed up.  


## Pre-requisite

1. Should have a backup Kafka Cluster - preferably in same verions as source. 
2. KafkaConnect Cluster not needed.
3. An external staging computer from where the backup / restore commands can be executed. Backed up files can be stored either in object storage or local block storage. 
4. The staging computer should have the 'oc' and 'kubectl' CLIs installed.    

## Backup from Source EventStreams

1. Create a local working folder in the staging computer.   
	> mkdir /Users/rajan/Downloads/copy_backup/topic-3p-2r  

2. From the OpenShift Console, check the location of the data.  
> View the \<ES-INSTANCE-NAME>-kafka-config configmap. [E.g. minimal-prod-kafka-config ].  
> Look for the entry:   
> log.dirs=/var/lib/kafka/data/kafka-log${STRIMZI\_BROKER\_ID}.  
> e.g. /var/lib/kafka/data/kafka-log0/

3. Check for files in the brokers.  

	`oc -n <NAMESPACE> exec -it <KAFKA-POD-NAME> -c kafka -- ls -lrt /var/lib/kafka/data/kafka-log<BROKER-ID>/`.  
	
	`e.g.  oc -n rajan-es1 exec -it minimal-prod-kafka-0 -c kafka -- ls -lrt /var/lib/kafka/data/kafka-log0/`
	
4. Determine which topics have to be copied. It is not recommended to copy the entire logs_dir as it may contain replica topics. No point copying replica topics as they are just a backup of original topics.   
5. For this example, we will replicate a topic called topic-3p-2r. The topic has 3 partitions and 2 replicas. We have to manually check each broker and determine which partition should be copied from which broker. This is a sample of how the distribution may look like:   

> Broker 0 - Partition 0 and 2.  
> Broker 1 - Partition 1 and 2.  
> Broker 2 - Partition 0 and 1.  

From here we can decide to copy as follows:

	Partition 0 from Broker 0
	Partition 1 from Broker 2
	Partition 2 from Broker 1

We just need to be sure to copy all 3 partitions. From which Broker we are copying it does not really matter. Leader partiton or follower partiton also does not matter. We will use 'oc rsync' to copy files. Check [here](https://docs.openshift.com/container-platform/4.10/nodes/containers/nodes-containers-copying-files.html) for full reference of 'oc rsync'.     

6. Copy the respectve partitions files of the topic from the respective brokers.   

Command:
`oc -n <name-space> rsync <POD-NAME>:/var/lib/kafka/data/kafka-log<BROKER-ID>/topic-3p-2r-<PARTITION-ID> /Users/rajan/Downloads/copy_backup/topic-3p-2r/  `.  

Partiton 0 from Broker 0.  
`oc -n rajan-es1 rsync minimal-prod-kafka-0:/var/lib/kafka/data/kafka-log0/topic-3p-2r-0 /Users/rajan/Downloads/copy_backup/topic-3p-2r/  `.  

Partition 1 from Broker 2.  
`oc -n rajan-es1 rsync minimal-prod-kafka-2:/var/lib/kafka/data/kafka-log2/topic-3p-2r-1 /Users/rajan/Downloads/copy_backup/topic-3p-2r/`.  

Partiton 2 frmo Broker 1.   
`oc -n rajan-es1 rsync minimal-prod-kafka-1:/var/lib/kafka/data/kafka-log1/topic-3p-2r-2 /Users/rajan/Downloads/copy_backup/topic-3p-2r/`.  

7. Check the destination. There should be 3 folders (that represents each partition) and multiple files in these folders.   

		Natarajans-MacBook-Pro-2:topic-3p-2r rajan$ pwd
		/Users/rajan/Downloads/copy_backup/topic-3p-2r
		Natarajans-MacBook-Pro-2:topic-3p-2r rajan$ ls -lta
		total 0
		drwxr-sr-x  7 rajan  staff  224 Feb 22 16:29 topic-3p-2r-2
		drwxr-sr-x  5 rajan  staff  160 Feb 22 16:29 .
		drwxr-sr-x  7 rajan  staff  224 Feb 22 16:29 topic-3p-2r-1
		drwxr-sr-x  7 rajan  staff  224 Feb 22 16:22 topic-3p-2r-0


## Restore to Target EventStreams

1. Now we will restore the copied files into another EventStreams. You need to have a namnespace created in the target cluster.  

		oc new-project <project-name>    
		E.g. oc new-project rajan-es1-dr

2. Before copying the partitons to the target cluster, we need to decide on which partiton should go into which broker. It is NOT necessary to match the partitions and brokers as they were in the source ES.  In fact, technically, it is possible to restore all 3 partitions in the same broker. For our testing, we will do the following:

		Partition 0 in Broker 0
		Partition 1 in Broker 1
		Partition 2 in Broker 2

3. Copy files to target cluster.  

Command:   

`oc -n <NAMESPACE> rsync /Users/rajan/Downloads/copy_backup/topic-3p-2r/topic-3p-2r-<PARTITION-ID> <POD-NAME>:/var/lib/kafka/data/kafka-log<BROKER-ID>/`

Partition 0 in Broker 0
`oc -n rajan-es1-dr rsync /Users/rajan/Downloads/copy_backup/topic-3p-2r/topic-3p-2r-0 my-dr-system-kafka-0:/var/lib/kafka/data/kafka-log0/`

Partition 1 in Broker 1
`oc -n rajan-es1-dr rsync /Users/rajan/Downloads/copy_backup/topic-3p-2r/topic-3p-2r-1 my-dr-system-kafka-1:/var/lib/kafka/data/kafka-log1/`

Partition 2 in Broker 2
`oc -n rajan-es1-dr rsync /Users/rajan/Downloads/copy_backup/topic-3p-2r/topic-3p-2r-2 my-dr-system-kafka-2:/var/lib/kafka/data/kafka-log2/`

4. Let the target Kafka know about the new topic and it's partition location. This is an important step where we will have to notify kafka about the mapping of the partition and broker.   

Command:
`oc -n <NAME-SPACE> exec -ti <POD Name> -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic topic-3p-2r  --replica-assignment 0:1:2,1:2:0,2:0:1 --config retention.ms=-1`.  

> 	Explanation of the command:   
> 	**replica-assignment** - to point the leader and replica for each partition. So, the first "0:1:2" represents Partition 0. Leader is Broker 0 where else, 1 and 2 are replicas. So, here we are defining 3 replicas for the topic eventhough there were only 2 replicas in the source kafka.   
> 	**config retention** - this is the retention period of the messages in topic. A '-1 ' means, no expiry period. This is to ensure older messages do not get deleted.   
> 

This is how the command looks like:   
`oc -n rajan-es1-dr exec -ti my-dr-system-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic topic-3p-2r  --replica-assignment 0:1:2,1:2:0,2:0:1 --config retention.ms=-1`.  

You shoud get a confirmation that the topic was created.   
	
5. You can now check the Target Kafka and ensure the messages are properly restored. For example, you can check the last offset of the source and target Kafka.    



