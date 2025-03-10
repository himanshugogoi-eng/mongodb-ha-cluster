# MongoDB High Availability (HA) Cluster Setup

## Overview
This documentation outlines the setup of a high-availability (HA) MongoDB cluster with automatic failover. The cluster consists of multiple replica set nodes(3), ensuring redundancy and minimal downtime.

## Architecture
The HA cluster is designed with the following components:

*Primary Node: Handles all write operations.

*Secondary Nodes: Replicates data from the primary node.

*Load Balancer (Optional): Directs traffic based on availability.

## Prerequisites
*Bare metal or cloud servers running Red Hat Enterprise Linux 9.5.

*MongoDB version: 8.0.5 Enterprise

*SSH access and sudo privileges.(Passwordless ssh enabled between the nodes.)

*Firewall rules allowing communication between nodes.

## Step-by-Step Setup
1. Install MongoDB on All Nodes. Add the repository to the repo file as obtained from the mongodb official site.	
```bash
	sudo dnf install -y mongodb-enterprise
```

3. With authentication disabled in the configuration file create an admin user for mongodb.
First stop the mongod service and start it with --no-auth option:
 ```bash
	sudo mongod --port 27017 --dbpath /var/lib/mongo --noauth
 ```	
4. Create the admin user in mongodb:
```javascript	
 	 use admin;
	 db.createUser(
	{
		user: "superAdmin",
		pwd: "Admin@123!",
		roles: [ { role: "root", db: "admin" } ]	
	}
	);
 ```	
5. Now restart the mongod service with authentication on by setting it in the configuration file.

6. Create a cluster admin user for the cluster as well in the same way.

7. Set hostnames on all the three nodes as per your choice. Here I am using the following hostnames:
	```
           172.31.4.37 mongo-primary
	   172.31.6.134 mongo-secondary-1
	   172.31.7.125 mongo-secondary-2
	```
8. Create a key file for authentication and distribute it among the servers.
```bash	
 	# mkdir /etc/mongodb
	# openssl rand -base64 756 > /etc/mongodb/keyfile
	# chmod 400 /etc/mongodb/keyfile
	# chown mongod:mongod /etc/mongodb/keyfile
	# rsync -avzP /etc/mongodb/keyfile mongo-secondary-1:/etc/mongodb/
	# rsync -avzP /etc/mongodb/keyfile mongo-secondary-2:/etc/mongodb/
```
9. Configure Replica Set

Edit MongoDB configuration file (/etc/mongod.conf) on all nodes:
```bash	
  vi /etc/mongod.conf
```
And configure the following:
```yaml	
 net:
	  port: 27017
	  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
	security:
	   authorization: enabled
	   keyFile: /etc/mongodb/keyfile

	replication:
	   replSetName: "rs0"
```
Restart MongoDB service:
```bash
sudo systemctl restart mongod
```
9. Initiate the Replica Set

On the primary node, run the MongoDB shell:
```bash
mongosh --host mongo-primary
```
Initiate the replica set:
```javascript
	rs.initiate({
  	_id: "rs0",
  	members: [
    	{_id: 0, host: "mongo-primary:27017" },
    	{ _id: 1, host: "mongo-secondary-1:27017" },
    	{ _id: 2, host: "mongo-secondary-2:27017" }
  		]
	});
```
10. Verify the Replica Set Status
```javascript
 	rs.status();
```

## Advanced Configuration Considerations
### Read Preference Modes
	Configure application connection strings for optimal read distribution:
```bash
	mongodb://appUser:AppPassword456!@mongo-primary:27017,mongo-secondary-1:27017,mongo-secondary-2:27017/applicationDB?replicaSet=rs0&readPreference=secondaryPreferred
```	
### Write Concern Levels
	Ensure data durability with appropriate write concerns:
 ```javascript
 	db.importantColl.insert(
	{ criticalData: "value" },
	{ writeConcern: { w: "majority", wtimeout: 5000 } }
	)
```

## Monitoring & Maintenance
Check logs: 
```bash
	sudo journalctl -u mongod --no-pager | tail
```
Monitor replica set: 
```javascript
	rs.status();
```
Backup: Use mongodump for periodic backups.

Disaster Recovery Strategy
Oplog Maintenance
Size the oplog appropriately for recovery windows:
```javascript
	db.adminCommand({replSetResizeOplog: 1, size: 204800});
```

### Backup Procedures
MongoDB Consistent Snapshots:
```bash
	mongodump --host rs0/mongo-primary:27017,mongo-secondary-1:27017 --ssl --authenticationDatabase admin --username backupUser --archive --oplog > backup-$(date +%F).archive
```
Restoration Process
```bash
	mongorestore --host rs0/mongo-new-primary:27017 --ssl --authenticationDatabase admin --username restoreUser --oplogReplay --archive=backup-2025-02-21.archive
```

### Performance Optimization
Index Management
Create compound indexes for common query patterns:
```javascript
	db.orders.createIndex({ customerId: 1, orderDate: -1 });
```

Sharding Considerations
For datasets exceeding 5TB:
```javascript
sh.addShard("rs0/mongo-primary:27017");
sh.enableSharding("bigDataDB");
sh.shardCollection("bigDataDB.logs", { _id: "hashed" });
```
## Conclusion
This comprehensive deployment strategy ensures 99.95% availability for MongoDB clusters through automated failover, robust replication, and proper security controls. The three-node architecture balances cost and reliability while providing a foundation for scaling to larger deployments. Regular maintenance including software updates, backup validations, and performance tuning remains crucial for long-term cluster health34.

For production environments, consider enhancing the base configuration with:

Dedicated monitoring nodes

Cross-AZ deployments for AWS/Azure

Queryable secondaries for analytics workloads

Bi-directional TLS between cluster members

By implementing these best practices, organizations achieve enterprise-grade database availability while maintaining the flexibility of MongoDB's document model

