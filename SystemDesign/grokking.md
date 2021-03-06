# Chapter 1-3
## Unique Key Generation either for sharding key/ unique url etc.

### Use separate Key Generation Service
- We can have a standalone Key Generation Service (KGS) that generates random six letter strings beforehand and stores them in a database.  we won’t have to worry about duplications or collisions. KGS will make sure all the keys inserted into key-DB are unique.

#### Concurrency Issues Solution
- KGS's database can use two tables to store keys: one for keys that are not used yet, and one for all the used keys. As soon as KGS gives keys to one of the servers, it can move them to the used keys table. 
- When KGS's application server starts, it can load some keys into the memory from database and mark them as used in the database, this ensures all the KGS's application servers get a unique key (however when the service restarts we loose all the prefetched keys, shouldn't be a problem since we have usually billions of keys).
- Also within the same KGS's application server we might need some kind of a mutex lock on the keystoring in-memory datastructure, so not 2 request to the same KGS server gets the same key. 
- KGS can have multiple application servers to avoid single point of failure

### Use auto-increment key option
- becomes difficult to scale and synchronise between various database servers.
- Overflow can be an issue.
- One solution to synchronization can be, have 2 database, one for generating only even keys and one for odd keys to avoid single point of failure : Approach followed by Flicker.

### Append date or other useful data to the unique key, useful to be used as a sharding key. Approach by Instagram
- Use several thousands of logical shards which are mapped to few physical shards, this will help in scale in future.
- Instagram combines 3 values to generate the unique key, custom epoch time in milliseconds (41 bits - for 41 years) + user_id%logical_shard_id (13 bits) + auto_increment_value%1024(10 bits, 1024 unique values per millisecond per shard). [Link](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- By combining shard_id, we don't have to worry about synchronization issues

# Chapter - 4
## Drop Box (From [Video](https://www.youtube.com/watch?v=PE4gwstWhmc))
- Write heavy, almost 1:1.
- ACID Requirements
- S3 was where the data was stored and mysql had metadata.
- They separated upload, download functionality (moved to AWS) and the synching and webhosting functionality i.e metadata (onprem). 
- Moved from polling to notification servers, so that server is not bombed by the incoming polling requests for the updated data. Using long polling.
- Redundancy in servers.
- To make sure there are less db calls from block servers to mysql db (since repeated db calls across such large distances is really expensive), they did a simple REST call to the meta servers, which had the internal logic to do all those repeated db calls (faster since meta server is closer to mysql db than blockserver) (They could have also used stored procedures in mysql).
- Realised it is easier to use memcache that to implement complicated database sharding to scale dbs.
- Because of high consistency requirements, they had to deal with lots of issues in memcache and DBs.
- used standby load balancers for relaibility
- Block level deduplication, if 2 users upload same file the datastore should know they are the same.
  - Each file is divided into chunks and a hash of the chunk is calculated and if it is already the same as some other chunk, then it get's mapped to the same chunk/block in s3.
- Sharding based on user id, seems like a good choice but shared folders makes it hard.

### Metadata/Journalling
- Store all the changes related to the file in meta data, allows for versioning.
- Making journal_id as the primary key they were able to append to the journal table pretty fast, since they were appending based on incrementing journal_id.
- MySQL stores a varchar of length less than 255, better than the one of length more than 255.
- Declaring all fields as not null also reduces the space consumed.

### Primary Key
- They used a combination of namespace_id(identifies user/shared user space), latest(bool to denote if this is the latest version) and an incrementing jornal_id (for fast append based on ID order).
- latest field makes reads faster since we don't have to move over deleted or older versions of files to get the latest one for a namespace, but it makes writes slower, since writing things in dropbox led to shuffling things a while, since the older version file has to be moved over to the older portion of the index, so they later removed the latest field from primary key to optimize for writes since their read : write ratio was 1:1 almost.
- This simple change of primary key from just id to ns_id and id was able to do a massive improvement in performace.
- id is per namespace, so there is no issue of overflow.
![](res/dropbox_2.jpg)

### Sync
- If a file is modified to a small extent, only the delta's are kind of sent to the block server from the client using rsync.
### GIL Lock
- You can run, only 1 python thread, even with multiple cores.
### Distributed Database and Challenges
- Challenges include, how to perform distributed transaction?
### Scaling Notification Server
- Millions of clients connected to each Notification Servers.
- They added 2 level hierarchy to distribute connections to notification servers.



![](res/dropbox_1.jpg)


# Facebook Messenger

## From Internet
- Use Cassandra for faster writes. Make the (from-user, to-user) the partition key and the timestamp (DESC order by) your clustering key.
```sql
CREATE TABLE chat_messages (
    message_id uuid,
    from_user text,
    to_user text,
    body text,
    class text,
    time timeuuid,
    PRIMARY KEY ((from_user, to_user), time)
) WITH CLUSTERING ORDER BY (time ASC);
```
- The above approach however doesn't allow the user to delete the message, since delete in one user's view will delete it in other's view too. You can have flags to deal with this issue. ```visible_one and visible_two```

# Google Docs
- Use a queue to make sure the changes from various users come one after the other, each change to the server has a version number.
- Based one the version in the PUT request and the version of the doc in the server, you can perform operational transform. 
- All operations are either insert and delete.
- All user's cursor should be communicated using websockets. Version history, error messages.
- An append only change log, can be used to go back to versions.
- Markdown to persist the formatted text in the database. 
- Data is denormalized, document based storage.
- https://www.youtube.com/watch?v=3MsWDQJmYOE

# Google Search
## Crawler
## Page Rank
## Index

# TikTok
- CDN, News feed(Same with instagram push pull)
- S3, Meta data use SQL

# Type ahead


# Facebook Messenger 
- unlike whatsapp, we don't delete the message and there is no encryption decryption.

## API Specs
- session_id initiateDirectChatSession(API_KEY, user_id_1, user_id_2, handshake_info)
- message_id sendMessage(API_KEY, session_id,msg)
- readNewMessages()
- inititateGroupChatSession(API_KEY, group_info, user_id)
- addUserToGroup()
- removeUserFromGroup()

## High Level Architecture
- All the microservices can have multiple instances of application server inside a docker container. These application servers are stateless.
- Each of these services have a specific DB with the distributed caching layer.
### Routing Service
- Persistent connection with the user
- Proxy/Gateway service
- Websocket connection with the user
- ????How does the client know which service to connect to????

### User Service
- User table will be sharded by the user id.
### User Registration
- Generates a unique code that the user gets in the mobile
- Mobile number can be the unique key.
- User registration table is also present to track the OTP, it's schema is just a user_id with a user_code.
- Each entry in user registration service will only be alive for say 15mins.
- Talks with the SMS Gateway service for OTP.

### Group Service
- Similar to other services, LB, Distributed Cache and database
- There will be 2 major tables one is Group Table
  - (PK: GroupID), Name, creationTime, memberCount.
  - Sharded by GroupId
- Other table is the group membership table
  - (PK : UserId, GroupId), createTime
  - There will also be an index on the userId.
  - Sharded by GroupId
- Since both these are sharded by the same key, so the info about the entire group will be in the same machine and there can be joins.

#### ?????Local Index vs Global Index on UserId for Group Service.???
  - ???Since the groups are sharded by group Id, if I want to find all groups that a user is part of, won't that require me to go through all shards?
  - ???I guess in this case it is better to have global index for the 2ndary index of the table (the userid) and sharded also by userid. See discussion in 17:40. Check global index and jump to the shard with corresponding group Id.

### Session Service
- Keeps track of all the private and group session.
- There will be a session table
  - sessionId(PrimaryKey), creationDate, Type(group/private)
  - sessionId can be a combination of userId of both the user in private and userId and groupID in group session.

- There will also be a session_message table
  - sessionId and timestamp (Combination will be Primary Key)
  - createtime, type(direct/group), senderId, recieverId, data are the other columns.
  - Based on the timestamp, it will be determined which message came first and which came second.
  - Timestamp can be a combination of epochtime and a counter, since the epochtime has the precision of only one second.
    - ????Doubt?? how to make sure the increment is consistent without using transaction. A queue? maybe a async communication.????/
    - Maybe a row level lock on the session only.
- Both the above tables will be sharded by the SessionId. So for a group/session we just hit a single partition for easier scaling.
- We can treat the message reciepts (read/ delivered) as another message itself and give it a type.
  -  you save the status of when a message seen in the same table as the message themselves. I guess the msg column is useless because you can conclude it from the type. So you can store null there.
  -  owever, there will be a lot of nulls in this raw = waste of disk space = worse performance in the long run. Maybe it would be better to save it in another table.
  -  Also why the status message is also stored in the same table because it avoids querying multiple tables and also the data field will not be null. For status message the data field will contain information about the message for which the status message is sent (like message id, status like delivered/seen etc.)


### Fan Out Service.
- Will send the message to the reciever based on the type of message
  - If the message is of type private then it directly talks to the routing service and sends the message.
  - If the message is of type group, then it talks to the group service to fetch all the users inside the group and then sends the request to the routing service to forward the message.
- In both the above cases, if the user has not opened the app or is offline, then the message is routed to the push notification service. 

### Push Notification Service
- There is a daemon service, inside the Operating system which talks to the push notification servers.
- The whatsapp push notification service will send the message to the android servers with a id_token
- The android server will forward the message to the daemon service in the phone.
#### types of 3rd party notification servers.
- Apple Push Notification Service (APN)
- Google Cloud Messaging/ Firebase Cloud Messaging (FCM)
- Windows Notification Servers (WNS)
- AWS IOT Core





### See this 
that's not true, first of all, seqId can be guaranteed to be incrementally unique, I will come to that later.Before that, considering you have different senders and tons of services and also network delay, say user A send two msgs to user B, msg1 first then msg 2, if you are using server time(monotonic) you cannot solve network delay which means if for some reason msg 2 reach your server before msg 1, you will display msg 2 then msg 1 which is not right, this can happen in group chat too. For your question on seqId, the idea is each user will have an individual 64-bit sequence id which is allocated by seqSvr which I've mentioned in first comment, it will guarantee this seqId in ascending order, you can think it as a current version number, user will apply for new version number frequently(for every msg sending) and also update his/her cur version number whenever sync with server, the seqSrv will maintain each user's cur_version and also a max_version which is pre-allocated, once user's cur_version reaches max_version, seqSvr will increase it by a step(lets say 10000), for instance, initially user A is assigned a seq Id 50, and a max_Id 10000, user A send a msg, the msg will be send along with the cur_version 50 then seqSvr update user A's seq Id to 51, so on and so forth, until it reaches 10000, then seqSvr update user's max_version to 20000. So to speak, seqId is not only monotonically increasing within a session, it monotonically increasing life long. One more optimization you can think of is how to reduce the update for billions of user, because you will need to persist the seq and max_version, one thing you can do is group several users so they can have a common max_version, whoever reaches it will trigger the update, and we don't have to maintain cur_seq, we only update max_version when it reaches threshold, this can significantly reduce the cost(assume you have 10 million qps, there is no such disk can handle this I believe, correct me if I'm wrong, by doing this it will become 10^3, which is okay), also benefit recover process if seqSvr failed. Btw, this approach has been used in real world. There are many issues with timestamp approach. Lets talk something about your counter approach, question 1: how do you make it unique in different replicas? you have to make this increment in a single server first then copy to other replicas, right? then it is a single point failure. question 2: if we have billions of active users or 10 million qps, how you can handle these counter updates to database? how long will it take to recover a failure node? how can you reduce the recovery time? question 3: how can you solve network delay? like the scenario I described.



# Security

## Symmetric Key
- AES

## Asymmetric Key
- For exchanging Symmetric Keys
- Can be used for (encryption decryption) and also (sign, verify)
- Usually to send a message from A to B.
  - The message is first encrypted using B's public key and the resulting encrypted message also signed using A's private key.
  - Now in B's side, B can verify that the message was indeed sent from A by verifying using A's public key and then decrypt the message using B's private key.


## Certificate
- To make sure that the public key that each party uses is valid, we sign the public key with the private key of the Certificate Authority.
- Certificate Authorities are trusted by most browsers and come preinstalled.

# To Read
- https://instagram-engineering.com/handling-growth-with-postgres-5-tips-from-instagram-d5d7e7ffdfcb
- https://instagram-engineering.com/search-architecture-eeb34a936d3a
