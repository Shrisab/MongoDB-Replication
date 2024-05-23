# MongoDB-Replication
# Cluster to cluster sync of MongoDB instances using mongosync.

# Purpose
Replicating database using mongosync using three EC2 instances as replica set.

# Introduction
Mongosync is a powerful tool provided by MongoDB to facilitate the synchronization of data between MongoDB clusters. It allows you to efficiently copy data from one MongoDB deployment to another, making it an essential tool for tasks such as migration, backup, and disaster recovery.

# Features:
● Efficient data synchronization between MongoDB clusters.
● Support for both one-time and continuous synchronization.
● Flexible configuration options to customize synchronization behavior.
● Ability to resume synchronization from the point of interruption.
● Integration with MongoDB Atlas and self-managed MongoDB deployments.
● MongoDB offers comprehensive documentation, tutorials, and community support for mongosync which enables users to leverage its full capabilities effectively.

# Procedure
Step 1 : Launch EC2 instances
	One instance each for replica set member node
	An instance for mongos process which will connect our client ( MongoDB compass to our sharded MongoDB cluster)
	
Step 2 : Configure security group to all the inbound traffic/connection on the mongodb server port

Step 3 : Install MongoDB (on all AWS EC2 Ubuntu instances)

	Import the public key used by the package management system.
		wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -

	Create a list file for MongoDB
		echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list

	Reload local package database
		sudo apt-get update
		
	Install the MongoDB packages
		sudo apt-get install -y mongodb-org
		
		Start MongoDB
			sudo systemctl start mongod

		Verify that MongoDB has started successfully.
			sudo systemctl status mongod
		
		You can optionally ensure that MongoDB will start following a system reboot by issuing the following command:
			sudo systemctl enable mongod
		
		Stop MongoDB
			sudo systemctl stop mongod
		
		Restart MongoDB.
			sudo systemctl restart mongod
			
		Begin using MongoDB
			mongosh 
			
	Uninstall MongoDB Community Edition
		Stop MongoDB	
			sudo service mongod stop
		
		Remove Packages
			sudo apt-get purge mongodb-org*
		
		Remove Data Directories
			sudo rm -r /var/log/mongodb
			sudo rm -r /var/lib/mongodb
			
Replication setup (Ubuntu AWS EC2 instance):
	
	ps -aef | grep mongo 
	
	create directories on all the EC2 instances
		mkdir -p replicaset/member

	Start mongoDB with the following command on every EC2 instance
		nohup mongod --port 28041 --bind_ip localhost,ec2-3-86-210-11.compute-1.amazonaws.com --replSet replica_demo --dbpath replicaset/member &
		
		nohup mongod --port 28042 --bind_ip localhost,ec2-3-82-236-231.compute-1.amazonaws.com --replSet replica_demo --dbpath replicaset/member &
		 
		nohup mongod --port 28043 --bind_ip localhost,ec2-44-201-173-85.compute-1.amazonaws.com --replSet replica_demo --dbpath replicaset/member &
		
		mongosh --host 3.86.210.11  --port 28041
		
		rsconf = {
				  _id: "replica_demo",
				  members: [
					{
					 _id: 0,
					 host: "3.86.210.11:28041"
					},
					{
					 _id: 1,
					 host: "3.82.236.231:28042"
					},
					{
					 _id: 2,
					 host: "44.201.173.85:28043"
					}
				   ]
				}
		
		rs.initiate(rsconf)
		rs.status()		
		rs.conf()
		
		Add an arbiter
			Create a new EC2 instance. This can be setup over application server or some other server as well. 
			mkdir -p replicaset/member
			nohup mongod --port 27017 --bind_ip localhost,ec2-44-202-164-113.compute-1.amazonaws.com --replSet replica_demo --dbpath replicaset/member &
			
			rs.addArb("ec2-44-202-164-113.compute-1.amazonaws.com:27017")
			rs.remove("ec2-44-202-164-113.compute-1.amazonaws.com:27017")	
				OR
			rs.addArb("44.202.164.113:27017")
			rs.remove("44.202.164.113:27017")
			
			MongoServerError: Reconfig attempted to install a config that would change the implicit default write concern. Use the setDefaultRWConcern command to set a cluster-wide write concern and try the reconfig again.
			
			db.adminCommand({
			  setDefaultRWConcern: 1,
			  defaultWriteConcern: { w: 2 }
			})

		
		Add a new replica set member :  44.202.164.113
			rs.add( { host: "ec2-44-202-164-113.compute-1.amazonaws.com:27017" } )
			rs.remove("ec2-44-202-164-113.compute-1.amazonaws.com:27017")

		
		To rename the host name:
		cfg = rs.conf()
		cfg.members[0].host = "localhost,ec2-54-187-25-38.us-west-2.compute.amazonaws.com:28041"		
		rs.reconfig(cfg)

# Mongosync Limitations:
● The minimum supported server version is MongoDB 6.0.
● The source and destination clusters must have the same release version.
● The minimum supported Feature Compatibility Version is 6.0.
● The source and destination clusters must have the same Feature Compatibility Version.
● The destination cluster must be empty.
● mongosync does not validate that the clusters or the environment are properly configured.
● Other clients must not write to the destination cluster while mongosync is running.
● If write blocking is disabled, the client must prevent writes to the source cluster before starting the commit process.

# Conclusion:
In conclusion, mongosync is a valuable tool for synchronizing data between MongoDB clusters, offering flexibility, efficiency, and reliability.
