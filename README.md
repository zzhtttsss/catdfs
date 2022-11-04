# CatDFS

[ 中文 ](README.zh-CN.md)

CatDFS is a open source lightweight distributed file system implemented in Golang. It references the design of
[《The Google File System》](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/gfs-sosp2003.pdf)and [HDFS](https://github.com/apache/hadoop).

<img src="./document/architecture.png" width="750" title="Architecture of CatDFS"/>

CatDFS includes four subprojects:
* [master](https://github.com/zzhtttsss/tinydfs-master): master, the brain of the system. It is responsible for
  managing chunkservers and metadata, similar to the NameNode in HDFS.
* [chunkserver](https://github.com/zzhtttsss/tinydfs-chunkserver): chunkserver, the storage node of the system. 
  It is responsible for storing files, similar to the DataNode in HDFS.
* [client](https://github.com/zzhtttsss/tinydfs-client): client, the user interacts with the file system through it.
* [base](https://github.com/zzhtttsss/tinydfs-base): base, contains common methods, constants and protocol files 
  for each subproject, and each subproject depends on it.

As a distributed file system, CatDFS mainly has the following features:
- **Abundant File operations**——Include upload file (add), download file (get), move file (move), delete file (remove), 
  get file information(stat), print directory (list), rename file (rename), and future append writes (append) will be 
  supported.
- **High reliability**——Files are stored in different chunkservers with multiple replica placement strategy, 
  and the number of replicas can be adjusted as a parameter.
- **High availability**——Master can be deployed in clusters for avoiding the single point of failure, and the raft 
  algorithm is used to ensure metadata consistency. As long as more than half of the master nodes are available, the 
  system can still work.
- **Shrinkage management**——When a chunkserver crashes, the system will perform a shrinkage operation, and transfer the 
  files stored on the crashed chunkserver to other chunkservers according to the shrinkage strategy to ensure that no copies
  are lost.
- **Expansion management**——Users can add chunkservers at any time, and the system will transfer files on other chunkservers
  according to expansion strategy.
- **Load balance**——When the user uploads files or the system shrinks and expands, the system will find the optimal 
  strategy to select the appropriate chunkserver to place the file, so that the disk usage of each chunkserver is 
  basically balanced.
- **Crash recovery**——Both master and chunkserver can be directly restarted and added to the system without configuration
  after crashing, and the information(metadata or chunks) stored on them will not be lost.
- **System monitoring**——Use `Cadvisor`+`Prometheus`+`Grafana` to visually monitor various indicators of the system.

As a project suitable for noob, CatDFS mainly has the following characteristics:
- **Complete functional features**——It implements most of the functions and features required by a distributed file system,
  which is helpful to understand and learn the distributed system and related dependent components.
- **Simple system architecture**——Use the simplest structure to build the system so that people can learn it quickly.
- **Clear design ideas**——provide complete design documents, including the design of various metadata and mechanisms, 
  so as to quickly master the design principles of the system.
- **Detailed comments**——Most functions and attributes have detailed comments to help you understand the functions and
  attributes.

<!-- TOC -->
* [CatDFS](#catdfs)
  * [Background](#background)
  * [Install](#install)
  * [Example](#example)
  * [Design](#design)
    * [Overall Structure](#overall-structure)
      * [Master](#master)
      * [Chunkserver](#chunkserver)
      * [Client](#client)
    * [Metadata](#metadata)
      * [Chunk Metadata](#chunk-metadata)
      * [DataNode Metadata](#datanode-metadata)
      * [File Tree](#file-tree)
    * [High Availability](#high-availability)
      * [Switch Leader](#switch-leader)
      * [Metadata Persistence](#metadata-persistence)
      * [Crash Recovery](#crash-recovery)
      * [Read-write Separation](#read-write-separation)
    * [File Transfer](#file-transfer)
      * [GRPC Streaming Transmission](#grpc-streaming-transmission)
      * [Transmission-IO Separation](#transmission-io-separation)
      * [Pipeline](#pipeline)
      * [Error Handling](#error-handling)
    * [Shrinkage](#shrinkage)
    * [Expansion](#expansion)
  * [Maintainers](#maintainers)
  * [License](#license)
<!-- TOC -->

## Background

CatDFS is mainly independently designed and implemented from scratch by two master students (who are also noob software 
engineers)[@zzhtttsss](https://github.com/zzhtttsss) and [@DividedMoon](https://github.com/dividedmoon). Our purpose is 
mainly to exercise our ability to independently design and implement projects, and to be familiar with distributed systems,
Raft algorithms and various dependent components. This is the first time we released an independently implemented project
on GitHub, and we are also noobs. So during the development process, the solutions to problems encountered are limited, 
and there are still many bugs or bad design in the CatDFS. Feel free to make suggestions and questions about CatDFS. 
We hope CatDFS can help others to learn the distributed file system, and we will continue improving CatDFS in the future~

To simulate a distributed environment, we use `Docker` for testing. In `Docker compose`, we deploy `3` `master`, 
`5` `chunkserver`, `Ectd` as a component for service registration and discovery, `Cadvisor`+`Prometheus`+`Grafana` 
as visualization monitored components.

## Install

1. Compile each module into a `docker` image, in the directory of each module:

```bash
docker build -t [name] .
```

2. Run `docker compose`：
```bash
docker compose -f [compose.yaml] up -d
```

## Example

In progress...

## Design

### Overall Structure

CatDFS mainly consists of three parts: Master, Chunkserver and Client. GRPC is used for communication between each part.

#### Master

Master is the brain of the whole system and is responsible for maintaining the metadata of the system (mainly including 
Chunk information, Chunkserver information and the file tree). In order to ensure high availability, the Master adopts a
multi-node deployment strategy and uses the Raft algorithm (implemented through [hashicorp/raft](https://github.com/hashicorp/raft))
to ensure the metadata consistency among the Master's multiple nodes. And use the log persistence method that implemented
by [hashicorp/raft](https://github.com/hashicorp/raft) to do metadata persistence. All Master nodes will add their own 
addresses to Etcd to ensure Clients and Chunkservers can find them. The Master synchronizes information with the 
Chunkserver when receiving the heartbeat of the Chunkserver(refreshes the metadata, and assigns file sending tasks) 
instead of sending commands to the Chunkserver actively. The Master will also handle the Client's request to find or 
modify the metadata according to the user's request.

#### Chunkserver

Chunkserver is the storage node of the system and is responsible for actually storing files uploaded by users. However, 
the Chunkserver does not store the whole file but chunks of a fixed size (64MB). Each file will be cut into chunks of 
the same size, and each Chunk will be stored in multiple Chunkservers according to the number of replica set by the user.
Chunkserver will periodically send heartbeat to Master, and exchange information with Master through heartbeat. The 
Chunkserver and other Chunkservers and Clients will transmit and receive Chunks through pipelines.

#### Client

The Client is a client used by the user. The user sends various commands to the system through the client. Specifically,
the Client will request the Master to get or modify the metadata, establish a pipeline with the Chunkserver and send or 
receive the Chunk.

<img src="./document/architecture.png" width="750" title="Architecture of CatDFS"/>

### Metadata

The metadata of CatDFS consists of three parts: Chunk metadata, DataNode (ie Chunkserver) metadata and the file tree.

#### Chunk Metadata

Chunk metadata mainly includes `chunksMap` and `pendingChunkQueue`. The `chunksMap` contains all the Chunks stored in 
the current system to achieve O(1) search for Chunks. Two sets are used in each Chunk to store the DataNodes that currently
store the Chunk (`dataNodes`), and the DataNodes that have been allocated to store the Chunk but have not yet confirmed
the completion of the storage (`pendingDataNodes`) . `pendingChunkQueue` is a thread-safe queue, which contains all currently
missing chunks waiting for system allocation. The presence of one chunk in the queue means that the chunk currently lacks
a replica.

#### DataNode Metadata

DataNode metadata mainly includes `dataNodeMap` and `dataNodeHeap`. The `dataNodeMap` contains all the DataNodes (ie Chunkservers)
that have not yet died in the current system to achieve O(1) search for DataNodes. Each DataNode will use a set to record
the Chunks that the DataNode has (`Chunks`) , and use a map to record all the Chunk sending tasks that have not yet been 
confirmed to be completed (`FutureSendChunks`).

#### File Tree

The structure of the file tree is Trie, each node represents a directory or file, and each node stores its own parent node
and child node (via a map). 

<img src="./document/metadata.png" width="750"/>

### High Availability
The high availability of the Master of CatDFS mainly adopts the Master multi-node deployment strategy (based on 
[hashicorp/raft](https://github.com/hashicorp/raft)) whose metadata consistency is guaranteed by the Raft algorithm, and
uses Etcd to implement service registration and discovery.

#### Switch Leader 

When the leader node crashes or there is a network failure so that leader node can not keep in touch with other nodes in
the cluster for a certain period of time, the master cluster will re-elect the leader node and perform the leader switch.

The new Leader node will:
* Write its own address into the corresponding key of Etcd to replace the address of the old Leader node.
* Register an Observer to monitor the follower node state change event, so that the Follower node can be removed from the
  cluster in time after it dies.
* Monitor Chunkserver heartbeat in a goroutine.
* Consume `pendingChunkQueue` in a goroutine.

The old leader node will:
* Unregister the Observer that monitors the state change event of the follower node.
* Cancel the context of the goroutine monitoring the Chunkserver heartbeat and the goroutine consuming `pendingChunkQueue`
  to stop these two goroutines.
* Write its address to the Follower directory of Etcd.


#### Metadata Persistence

The metadata persistence of CatDFS utilizes the persistence mechanism of [hashicorp/raft](https://github.com/hashicorp/raft).
Specifically, each Master node will store all current log information (Log), cluster information and regularly save snapshots.
The Log contains changes to metadata, the cluster information contains the node status of the current cluster, and the 
snapshot contains all metadata information.

#### Crash Recovery

After a Master node crashes, it is removed from the master cluster and Etcd by the leader node. If the crashed node is a
leader node, a new leader node will be elected to complete the above operations.

When the Master node restarts, it will first read the snapshot and then re-execute each log after the snapshot to restore
the metadata. After that it will rejoin the master cluster and re-register its address in Etcd, and get the log from the
leader node to bring the metadata to the latest state. After completing the above steps, the Master node can provide 
services again.

#### Read-write Separation

Read-write separation means writing metadata from the leader node and reading metadata from the follower node. For the 
operations of reading metadata, such as `List` and `Stat`, we provide two modes, one is to read the metadata from the 
leader node (Latest), this mode can ensure that the latest data is read; the other is Randomly select a node from the 
follower node to read the metadata (Stale). In this mode, the metadata has a delay of about 50ms. Users can add parameters
to the command to set the mode.

### File Transfer

File transfer mainly uses the Pipeline mechanism, which will establish a pipeline between the Client and multiple 
Chunkservers or among multiple Chunkservers, and the data will be sent from sender to each node on the pipeline.

#### GRPC Streaming Transmission

Because the size of a chunk is 64mb, normal GRPC transmission is not suitable for such large files, so we use GRPC 
streaming transmission, which divides the whole chunk into multiple `piece` to transmit.

#### Transmission-IO Separation

Because IO is slow, frequent IO by the main thread will greatly affect the transmission efficiency of the pipline, so
the Chunkserver write the received piece to the hard disk in a goroutine, and the main thread is only responsible for 
receiving pieces, put pieces into the pipeline to give them the goroutine and send pieces to the next Chunkserver.

#### Pipeline

When Chunkserver A is called by a client or another Chunkserver via RPC, it will get the other Chunkservers that need to
receive the chunk from the metadata passed by the stream, and take the first Chunkserver B to call B's `transferChunk`
method via RPC to get a stream which A can send data to B by.

Assuming that now there is a client that needs to transfer a Chunk to Chunkserver A, Chunkserver B, and 
Chunkserver C, this design allows the client to call A's `transferChunk` method through RPC to establish a stream to 
transfer the chunk. A will also call B's `transferChunk` method. To send data to C, B will also establish a stream by 
calling the C's `transferChunk` method, so that the pipeline is established. When the client transmits a piece to A, 
A will immediately use the stream held by B to send the piece. And B will send it to C in the same way after receiving it.

<img src="./document/transfer.png" width="750"/>

#### Error Handling

Each Chunkserver in a pipeline can occur two kinds of errors: a file write error or a data transfer error. File write 
errors only affect a single node, while data transfer errors affect the node where the error occurred and all subsequent
nodes. In order to get all the nodes that failed to store chunks so that the master can make up for the missing replicas
later, each chunkserver returns an error list to the previous node in the pipeline after the transfer is complete. The 
list contains the node addresses of all nodes it found that have failed to transmit. The maintenance strategy of the list
is as follows:
* When the node has a file write error, it will add its own address to the list.
* When an data transfer error occurs when a node sends data to the next node, all node addresses after the node are added
  to the list.
* When the node receives data transfer error, directly stop receiving and sending data.
* After the node receives the error list returned by the next node, it will merge its current list with this list.

Through this strategy, the first node of the pipeline(ie sender) can get all the nodes in the pipeline that fail to store
Chunks, and count the number of nodes and return them to the Master through heartbeat. The Master will add this amount 
of the Chunk to the `pendingChunkQueue`, and then fill up the copy of the Chunk by consuming the `pendingChunkQueue`.

<img src="./document/transferError.png" width="750"/>

### Shrinkage

After the Master does not receive the heartbeat of a Chunkserver for a certain period of time, it will determine that the
Chunkserver is dead, triggering the shrinkage. The specific process is shown in the figure (the picture is so large that
Github cannot display it well, it is recommended to download and view):

<img src="./document/shrink.png" width="750"/>

### Expansion

In progress...


## Maintainers

[@zzhtttsss](https://github.com/zzhtttsss)

[@DividedMoon](https://github.com/dividedmoon)

## License

[GPL](LICENSE)
