## 1 Introduction

BFT (Byzantine Fault Tolerance) is a model of a distributed system that tolerates the degree of error tolerance. If a distributed system can tolerate any errors (these errors may include hardware errors, network congestion and latency, hacker attacks, node hacking ), We say that this system has reached Byzantine fault tolerance. Although the early 80s of the last century, lamport has proved the feasibility of Byzantine fault tolerance, but has not been a practical, efficient algorithm to achieve until castro and liskov published PBFT, where P said Pratical.

DPOS (Delegated Proof Of Stake) is a consensus mechanism based on the client's proof of entitlement, mainly used to achieve the consistency of distributed books. Although the actual application of the DPOS system, such as bitshares, crypti have not encountered a major security problem, but after our testing and analysis, it exists the theoretical loopholes, if hackers can easily use the system to cause bifurcation, Triggering the risk of double payments. That is to say that this system did not meet Byzantine fault tolerance.

The project has done two things

Analyze the vulnerability of the DPOS algorithm and simulate the simplest hacker attacks, resulting in network bifurcation
Introduce the PBFT algorithm to enhance the security aspects of DPOS to tolerate Byzantine errors
In order to simplify the model, the code does not achieve block chain persistence, signature authentication, block synchronization and other functions, only to achieve a basic network model, it can only be used for demonstration and teaching, temporarily can not be used for real projects.

## 2 DPOS analysis

The core of the consensus algorithm in the block chain is to solve three problems

Who will produce the block
When to generate block
How to verify the legitimacy of the block
DPOS is the answer

Determined by the current top-ranked list of principals and the current time offset
In regular intervals
Because you can determine the legal forging by the timestamp of the block, you can verify the legitimacy of the signature and the client's public key provided by the block, and verify that the transaction in the block and the connection to the previous block are omitted.
The pseudocode is shown below

for round i
    dlist_i = get N delegates sort by votes
    dlist_i = shuffle(dlist_i)
    loop
        slot = global_time_offset / block_interval
        pos = slot % N
        if delegates[pos] exists in this node
            generateBlock(keypair of delegates[pos])
        else
            skip
There are two main disadvantages of DPOS algorithm.

One is to use the timestamp on the number of clients to take the way to determine the current time slice forging, which increases the possibility of error, if the client server time drift (such as may be a network problem or not properly configured ntp service), it will cause the network of bifurcation, the specific reference here [ https://github.com/LiskHQ/lisk/issues/82 ]

The second is that the client's rights are too high, may be abused, because the DPOS is not like POW on the calculation of computing power, DPOS client forging blocks do not need to calculate the power, they can forge in the moment countless blocks, And sent to different network nodes, resulting in network bifurcation.

For the first question, DPOS did not have a good response, only hope that the client has a good server operation and maintenance experience, if they are a little careless, there will be cards, bifurcation risk. The second problem, DPOS mainly to deal with the client is a random order and the longest chain synchronization method. The longest chain synchronization is the most basic technology, there is no talk, we analyze the random sort.

To crypti, for example, crypti in the principal has 101, forging rate is 10s a, each round of the election cycle of about 16.8 minutes. The seed of the sorting algorithm for each period is invariant. Specific reference to the following code
```
function shuffle(height) {
  var truncDelegateList = [];
  for (var i = 0; i < 101; i++) {
    truncDelegateList.push(i);
  }
  var seedSource = String(Math.floor(height / 101) + (height % 101 > 0 ? 1 : 0));
  console.log(seedSource);
  var currentSeed = crypto.createHash('sha256').update(seedSource, 'utf8').digest();
  for (var i = 0, delCount = truncDelegateList.length; i < delCount; i++) {
    for (var x = 0; x < 4 && i < delCount; i++, x++) {
      var newIndex = currentSeed[x] % delCount;
      var b = truncDelegateList[newIndex];
      truncDelegateList[newIndex] = truncDelegateList[i];
      truncDelegateList[i] = b;
    }
    currentSeed = crypto.createHash('sha256').update(currentSeed).digest();
  }
  return truncDelegateList;
}```
That is, in this 16.8 minutes, 101 orders for the order of forging the client is OK, which gave the hacker a lot of operating space.

For example, the list of delegates after the sort is as follows

1,6,9,10,50,70,31,22,13,25
The hacker actually controls the node for

1,10,70,31,13,25
Hackers in the No. 1 node caused by the network after the bifurcation, due to the interval between several loyal nodes, bifurcation quickly by the longest chain synchronization mechanism to eliminate, but if the hackers at this time on the loyalty of these nodes initiated DDOS attacks, then He will be able to make their invasion of the original non-consecutive malicious nodes continue to produce blocks, that is to say the bifurcation will continue to six blocks, then the two bifurcated network transactions will be confirmed 6 Times, these transactions may include conflicting transactions. That is to say that hackers only need to control the six nodes, with DDOS can be 100% resulting in double payment.

## 3 PBFT

After joining PBFT, the first half of the DPOS algorithm does not change, that is, the list of clients and the sorting algorithm does not change.

The latter part of the change is the validation and persistence of the block. The verification of the block, no longer using a single signature verification, but the way the whole node votes, whenever the new block created, the loyal node does not immediately write it to the block chain, but wait for other nodes vote. When the number of votes more than a certain number before entering the implementation phase.

The algorithm assumes that the number of error nodes does not exceed f and the number of points n> = 3f + 1, then the system can guarantee the consistency of the block by satisfying the following two conditions

If a correct node accepts and executes a block, then all the correct nodes commit the same block
All the correct nodes either lag behind the longest chain, or the same as the longest chain, do not appear the same height but block hash different situation
The algorithm flow is as follows:

The current time piece of forging will pack the collected transactions into blocks and broadcast (this step is consistent with the previous algorithm)
If you have not received the block and have validated it, it broadcasts a prepare <h, d, s> message, where h is the height of the block, d is the summary of the block, s is the node signature
After receiving the prepare message, the node starts to accumulate the number of messages in memory. When the prepare message is received after more than f + 1 different nodes, the node enters the prepared state, and then a commit <h, d, s> message is broadcast
Each node receives more than 2f + 1 different node commit message, it is considered that the block has reached an agreement, into the committed state, and its persistence to the block chain database
The system in the first height of the block received h, start a timer, when the time expires, if not yet reached a consensus, to give up this consensus.
It should be noted that this algorithm is different from the algorithm in the PBFT paper. One is to cancel the state of commit-local, the other is no view changes in the process. Since PBFT was originally proposed primarily for a general CS architecture service system, the server responds to every request from the client, but in the block chain system, the block generation can be delayed until the next time And this delay is very common, which is essentially the same as the view change, each time slice is equivalent to a view, slot = time_offset / interval, the slot is equivalent to the view number.

As a result of the cancellation of the view changes, to achieve a consensus performance is also greatly improved. Suppose there are N nodes in the system, including the client node and the common node. The message flow using the gossip algorithm, a broadcast need to pass the message limit is N ^ 2, the corresponding time cost is O (logN). If the ordinary node only receives no forwarding, then N can be reduced to the total number of nodes of the client n, because the number of clients in the system for a certain period of time remain unchanged, you can think of a broadcast time cost is constant t. To confirm a block requires 3 rounds of broadcast, that is, the total time is 3t. block message size is set to B, the total message consumption is (B + 2b) N ^ 2.

The correctness of the algorithm is not proved here, we can refer to lamport and liskov the original paper, the project did not innovate any algorithm, but a classic PBFT algorithm to achieve, and slightly modified with the DPOS together. Demo results can see the results of the operation, which can be considered a proof.

## 4 demo instructions

installation

npm install
run
```
// 帮助
node main.js -h

// 使用pbft, 默认不使用
node main.js -p

// 模拟错误节点，-b后跟节点id列表，逗号分隔
node main.js -b 1,2,3

// 组合使用pbft，和错误节点
node main.js -b 1,2,3 -p
```
## 5 demo

First we use the default dpos algorithm to simulate a bifurcation attack

node main.js -b 10
Wait until the 10th node forged blocks, it will create two fork, and sent to a different node, you can see at a height of 4, it began to fork
```
fork on node: 10, height: 4, fork1: 58b1c8d429f7ed6d47bf6e7bead2139af420be453259ea0da42091ced3b28ed8, fork2: 61084a05844c436a36dc1f14ad151bda19ab3774aa15d8b1006cbe1dfb01b943
send fork1 to 2
send fork2 to 5
send fork1 to 6
send fork2 to 8
send fork1 to 9
node 0 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 1 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 2 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 3 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 4 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 5 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 6 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 7 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 8 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 9 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 10 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 11 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 12 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 13 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 14 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 15 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 16 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:58b1c8:10) -> 
node 17 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 18 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
node 19 (0:713304:0) -> (1:f76cf6:7) -> (2:f1d1bc:8) -> (3:cccd58:9) -> (4:61084a:10) -> 
```
Then, we open the pbft option and specify 4 "bad nodes"
```
node main.js -p -b 1,5,7,10
```
After several rounds of bifurcation attacks, we see all the normal nodes are the same, only a few bad nodes are inconsistent
```
node 0 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 1 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:faf2c9:5) -> (10:0ed227:10) -> 
node 2 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 3 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 4 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 5 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:faf2c9:5) -> 
node 6 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 7 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:faf2c9:5) -> (10:0b98f7:7) -> 
node 8 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 9 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 10 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:faf2c9:5) -> (10:0ed227:10) -> 
node 11 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 12 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 13 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 14 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 15 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 16 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 17 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 18 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
node 19 (0:713304:0) -> (1:bb9e59:17) -> (2:641aad:18) -> (3:e2614b:19) -> (4:ac5538:0) -> (5:c82859:1) -> (6:015639:2) -> (7:288ce7:3) -> (8:fdc189:4) -> (9:7eb1e1:6) -> (10:3d33b9:8) -> (11:851887:9) -> (12:37a10a:11) -> (13:7d0a62:12) -> (14:08376c:13) -> (15:7221d3:14) -> 
```
