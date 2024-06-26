# BP-31: BookKeeper Durability (Anchor)


## Motivation
Apache BookKeeper is transitioning into a full fledged distributed storage that can keep the data for long term. Durability is paramount to achieve the status of trusted store. This Anchor BP discusses many gaps and areas of improvement.  Each issue listed here will have another issue and this BP is expected to be updated when that issue is created.

## Durability Contract
1. **Maintain WQ copies all the time**. If any ledger falls into under replicated state, there needs to be an SLA on how quickly the replication levels can be brought back to normal levels.
2. **Enforce Placement Policy** strictly during write and replication.
3. **Protect the data** against corruption on the wire or at rest.

## Work Grouping (In the order of priority)
### Detect Durability Validation
First step is to understand the areas of durability breaches. Design metrics that record durability contract violations. 
* At the Creation: Validate durability contract when the ledger is being created
* At the Deletion: Validate durability contract when ledger is deleted
* During lifetime: Validate durability contract during the lifetime of the ledger.(periodic validator)
* During Read: IO or Checksum errors in the read path
### Delete Discipline
* Build a single delete choke point with stringent validations
* Archival bit in the metadata to assist Two phase Deletes
* Stateful/Explicit Deletes
### Metadata Recovery
* Metadata recovery tool to reconstruct the metadata if the metadata server gets wiped out. This tool need to make sure that the data is readable even if we can't get all the metadata (ex: ctime) back.

### Plug Durability Violations
Our first step is to identify durability viloations. That gives us the magnitude of the problem and areas that we need to focus. In this phase, fix high impact areas.
* Identify source of problems detected by the work we did in step-1 above (Detect Durability Validation)
* Rereplicate under replicated ledgers detected during write
* Rereplicate under replicated / corrupted ledgers detected during read
* Replicated under replicated ledgers identified by periodic validator.
### Durability Test
* Test plan, new tests and integrating it into CI pipeline. 
### Introduce bookie incarnation 
* Design/Implement bookie incarnation mechanism 
### End 2 End Checksum
* Efficient checksum implementation (crc32c?)
* Implement checksum validation on bookies in the write path. 
### Soft Deletes
* Design and implement soft delete feature
### BitRot detection
* Design and implement bitrot detection/correction.

## Durability Contract Violations 
### Write errors beyond AQ are ignored.
BK client library transparently corrects any write errors while writing to bookie by changing the ensemble. Take a case where `WQ:3 and AQ:2`. This works fine only if the write fails to the bookie before it gets 2 successful responses. But if the 3rd bookie write fails **after** 2 successful responses and the response sent to client, this error is logged and no immediate action is taken to bring up the replication of the entry.
This case **may not be**  detected by the auditor’s periodic ledger check. Given that we allow out of order write, that in the combination of 2 out of 3 to satisfy client, it is possible to have under replication in the middle of the ensemble entry. Hence ledgercheck is not going to find all under replication cases, on top of that,   periodic ledger check  is a complete sweep of the store, an very expensive and slow crawl hence defaulted to once a week run.

### Strict enforcement of placement policy 
The best effort placement policy increases the write availability but at the cost of durability. Due to this non-strict placement, BK can’t guarantee data availability when a fault domain (rack) is lost. This also makes rolling upgrade across fault domains more difficult/non possible. Need to enforce strict ensemble placement and fail the write if all WQ copies are not able to be placed across different fault domains.  Minor fix/enhancement if we agree to give placement higher priority than a successful write(availability)

The auditor re-replication uses client library to find a replacement bookie for each ledger in the lost bookie. But bookies are unaware of the ledger ensemble placement policy as this information is not part of metadata. 

### Detect and act on Ledger disk problems
While Auditor mechanism detects complete bookie crash, there is no mechanism to detect individual ledger disk errors. So if a ledger disk goes bad, bookie continues to run, and auditor can’t recognize under replication condition, until it runs the complete sweep, periodic ledger check. On the other hand bookie refuses to come up if it finds a bad disk, which is right thing to do. This is easy to fix, in the interleaved ledger manger bad disk handle.

### Checksum at bookies in the write path
Lack of checksum calculations on the write path makes the store not to detect any corruption at the source issues. Imagine NIC issues on the client. If data gets corrupted at the client NIC’s level it safely gets stored on bookies (for the lack of crc calculations in the write path). This is a silent corruption of all 3 copies.  For additional durability guarantees we can add checksum verification on bookies in the write path. Checksum calculations are cpu intensive and will add to the latency. But Java9 is adding native support for CRC32-C - A hardware assisted CRC calculation. We can consider adding this additional during JAVA-9 port after evaluating performance tradeoffs. 

### No repair in the read path
When a checksum error is detected, in addition to finding good replica, sfstore need to repair(replace with good one) bad replica too.


## Operations
### No bookie incarnation mechanism
A bookie `B1 at time t1` ; and same bookie `B1 at time t2` after bookie format are treated in the same way.
For this to cause any durability issues:
* Replication/Auditor mechanism is stopped or not running for some reason. (A stuck auditor will start a new one due to ZK)
* One of bookies(B1) went down (crash or something)
* B1’s Journal dir and all ledger dir got wiped.
* B1 came back to life as a fresh bookie
* Auditor is enabled  monitoring again

At this point auditor doesn’t have capability to know that the B1 in the cluster is not the same B1 that it used to be. Hence doesn’t consider it for under replication. This is a pathological scenario but we at least need to have a mechanism to identify and alert this scenario if not taking care of bookie incarnation issue.

## Enhancements
### Delete Choke Points
Every delete must go through single routine/path in the code and that needs to implement additional checks to perform physical delete.

### Archival bit in the metadata to assist Two phase Deletes
Main aim of this feature is to be as conservative as possible on the delete path. As explained in the stateful explicit deletes section, lack of ledgerId in the metadata means that ledger is deleted. A bug in the client code may erroneously delete the ledger. To protect from that, we want to introduce a archive/backedup bit. A separate backup/archival application can mark the bit after successfully backing up the ledger, and later on main client application will send the delete. If this feature is enabled, BK client will reject and throw an exception if it receives a delete request without archival/backed-up bit is not set. This protects the data from bugs and erroneous deletes.

### Stateful explicit deletes
Current bookkeeper deletes synchronously deletes the metadata in the zookeeper. Bookies implicitly assume that a particular ledger is deleted if it is not present in the metadata. This process has no crosscheck if the ledger is actually deleted. Any ZK corruption or loss of the ledger path znodes will make bookies to delete data on the disk. No cross check. Even bugs in bookie code which ‘determines’ if a ledger is present on the zk or not, may lead to data deletion. 

Right way to deal with this is to asynchronously delete metadata after each bookie explicitly checks that a particular ledger is deleted. This way each bookie explicitly checks the ‘delete state’ of the ledger before deleting on the disk data. One of the proposal is to move the deleted ledgers under /deleted/&lt;ledgerId&gt; other idea is to add a delete state, Open->Closed->Deleted.

As soon as we make the metadata deletions asynchronous, the immediate question is who will delete it?
Option-1: A centralized process like auditor will be responsible for deleting metadata after each bookie deletes on disk data.
Option-2: A decentralized, more complicated approach: Last bookie that deletes its on disk data, deletes the metadata too.
I am sure there can be more ideas. Any path will need a detailed design and need to consider many corner cases.

#### Obvious points to consider:
ZK as-is heavily loaded with BK metadata. Keeping these znodes around for more time ineeded puts more pressure on ZK.
If a bookie is down for long time, what would be the delete policy for the metadata?
There will be lots of corner case scenarios we need to deal with. For example: 
A bookie-1 hosting data for ledger-1  is down for long time
Ledger-1 data has been replicated to other bookies
Ledger-1 is deleted, and its data and metadata is cleared.
Now bookie-1 came back to life. Since our policy is ‘explicit state check delete’ bookie-1 can’t delete ledger-1 data as it can’t explicitly validate that the ledger-1 has been deleted. 
One possible solution: keep tombstones of deleted ledgers around for some duration. If a bookie is down for more than that duration, it needs to be decommissioned  and add as a new bookie. 
Enhance: Archival bit in the metadata to assist Two phase Deletes
Main aim of this feature is to be as conservative as possible on the delete path. As explained in the stateful explicit deletes section, lack of ledgerId in the metadata means that ledger is deleted. A bug in the client code may erroneously delete the ledger. To protect from that, we want to introduce a archive/backedup bit. A separate backup/archival application can mark the bit after successfully backing up the ledger, and later on main client application will send the delete. If this feature is enabled, BK client will reject and throw an exception if it receives a delete request without archival/backed-up bit is not set. This protects the data from bugs and erroneous deletes.

### Metadata recovery tool
In case zookkeper completely wiped we need a way to reconstruct enough metadata to read ledgers back. Currently metadata contains ensemble information which is critical for reading ledgers back, and also it has additional metadata like ctime and custom metadata. Every bookie has one index file per ledger and that has enough information to reconstruct the ensemble information so that the ledgers can be made readable. This tool can be built in two ways.
If ZK is completely wiped, reconstruct entire data from bookie index files.
If ZK is completely wiped, but snapshots are available, restore ZK from snapshots and built the delta from bookie index files.

### Bit Rot Detection (BP-24)
If the data stays on the disk for long time(years), it is possible to experience silent data degradation on the disk. In the current scenario we will not identify this until the data is read by the application.

### End to end checksum
Bookies never validate the payload checksum. If the client’s socket has issues, it might corrupt the data (at the source) and it won’t be detected until client reads it back. That will be too late as the original write was successful for the application. Use efficient checksum mechanisms and enforce checksum validations on the bookie’s write path. If checksum validation fails, write itself will fail and application will be notified. 


## Test strategy to validate durability
BK need to develop a comprehensive testing strategy to test and validate the store’s durability. Various methods and levels are tests are needed to gain confidence for deploying the store in production. Specific points are mentioned here and these are in addition to regular functional testing/validation.
### White box error injection
Introduce all possible errors in the write path, kick replication mechanism and make sure cluster reached desired replica levels.
Corrupt first readable copy and make sure that the corruption is detected on the read path, and ultimately read must succeed after trying second replica.
Corrupt packet after checksum calculation on the client and make sure that it is detected in the read path, and ultimately read fails as this is corruption at the source.
After a write make sure that the replica is distributed across fault zones.
Kill a bookie, make sure that the auditor detected and replicated all ledgers in that bookie according to allocation policy (across fault zones)
### Black box error injection (Chaos Monkey)
While keeping longevity testing which is doing continues IO to the store introduce following errors.
Kill random bookie and reads should continue.
Kill random bookies keeping minimum fault zones to satisfy AQ Quorum during write workload.
Simulate disk errors in random bookies and allow the bookie to go down and replication gets started.
Make sure that the cluster is running in full durable state through the tools and monitoring built.
