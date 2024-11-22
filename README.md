# mongo-sharded-project

This repository creates a basic sharded mongo database with a replication config set at four sharded clusters in a Docker multi-container setup. To create the sharded clusters, config cluster, and mongos node run

`docker-compose up -d`

The setup instructions below assume that you have the [mongosh CLI](https://www.mongodb.com/try/download/shell) already installed, however that is not necessary. Some alternatives include using [MongoDB Compass GUI](https://www.mongodb.com/try/download/compass) to connect into nodes and run commands, or using Docker's integrated terminal with a valid Mongo image. You must have the mongoimport tool that is exported as part of [MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools) to import the sample data in this repository discussed later.

For our mongos router to work properly, we must notify it of the config cluster used for replication and the sharded clusters which we will utilize for sharded collections. To do this, we network into the running shard containers with a MongoDB URI and run the following commands:

## Step 1: Initiate Replication Set

1. Connect to any Node in the config cluster  
   `mongosh mongodb://localhost:40001`
2. Run the replication set initiate command

   ```
   rs.initiate(
        {   _id: "cfgrs",
            configsvr: true,
            members: [
                { _id : 0, host : "cfgsvr1:27017" },
                { _id : 1, host : "cfgsvr2:27017" },
                { _id : 2, host : "cfgsvr3:27017" }
            ]
        }
    )
   ```

   NOTE: Useful command for verification: `rs.status()`. We mainly want to see that our independant nodes now have an elected PRIMARY node, with the others being SECONDARY and pointing to the PRIMARY master node.

## Step 2: Initialize Sharded Sets

1. Connect to any Node in the config cluster  
   `mongosh mongodb://localhost:50001`
2. Run the shard set initiate command

   ```
   rs.initiate(
        {
            _id: "shard1rs",
            members: [
                { _id : 0, host : "shard1svr1:27017" },
                { _id : 1, host : "shard1svr2:27017" },
                { _id : 2, host : "shard1svr3:27017" }
            ]
        }
    )
   ```

3. Repeat for the other three shards, on ports (50004, 50007, 50010). Replace shard1 with shard2, shard3, and shard4 respectively.

## Step 3: Add Shards to Router

1. Connect to the Mongos Router

   `mongosh mongodb://localhost:60000`

2. Map Shard Network

   ```
   sh.addShard("shard1rs/shard1svr1:27017,shard1svr2:27017,shard1svr3:27017")
   sh.addShard("shard2rs/shard2svr1:27017,shard2svr2:27017,shard2svr3:27017")
   sh.addShard("shard3rs/shard3svr1:27017,shard3svr2:27017,shard3svr3:27017")
   sh.addShard("shard4rs/shard4svr1:27017,shard4svr2:27017,shard4svr3:27017")
   ```

   NOTE: Useful command for verification: `sh.status()`. Useful to see balancing active, shard activations, shard keys, and how collections are stored on and across shards.

## Step 4: Create Sharded Collection and Data Primer Import

1. Connect to the Mongos Router

   `mongosh mongodb://localhost:60000`

2. Create Database

   `use csci5448`

3. Create our Collection to be Sharded

   `db.createCollection("sharded_demo")`

4. Shard the created Collection

   `sh.shardCollection("csci5448.sharded_demo", { "bikeid" : "hashed" } )`

5. Import the data to mongos, which will automatically shard the data across cluster (not in mongosh, use cmd/bash/etc.)

   ```
   mongoimport --uri "mongodb://localhost:60000" --db csci5448 --collection sharded_demo --file "./data/trips.json" --jsonArray
   ```

NOTE: Useful command `db.sharded_demo.getShardDistribution()` Useful to see the number of records split in node, bytes in nodes, hash ranges, etc. Great to see how much complexity is hidden away via the mongos facade.
