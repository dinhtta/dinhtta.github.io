---
layout: post
title: Consensus
---

In the last 3 months I finally came to face one of most important, yet most challenging (and feared) problem
in distributed systems, namely _distributed consensus_. First raised in 1980s, the problem of getting a set of
parties to agree on some value, remains an active area of research, with papers after papers appearing in
top-tier systems conferences like SIGCOMM, NSDI, SOSP, OSDI. One may remark at the fact that researchers in
the field have not reached a consensus on this decades-old consensus problem. Even without the human
irrationality in the loop, the range of subtlety and unpredictability needed to be considered in solving
consensus can perhaps be rivalled only by the human political systems. After all, there is no consensus on
what is the best political system, and we as a race have worked on it since forever (and if this year is of
any indication, we are failing specularly).  

Like an enormous elephant in the room that one may try his best (but in vain) to ignore, the latest
manifestation of distributed consensus is in the form of _blockchain_. At the heart of Bitcoin, Ethereum,
Hyperledger, etc. is the protocol that ensures Byzantine fault tolerance: some participants are malicious and
powerful, but the remaining honest participants (the majority) can still perform useful and correct work. As
blockchain systems are gaining massive interest from the industry, this complex problem is again demanding
attention. 

As noted in [another post](../flpcap) I remember next to nothing of what was taught in the Distributed System
course during my under graduate. But there are concepts that at the time I knew distinctively that I would
forget. In fact, just recalling them by words bring up fields of darkness. Among them are _leader election_,
_Byzantine agreement_, which I remember spending hours remembering the _details_ of the long, inscrutable
protocols without fully understand their relevant. To be fair, back then in 2005, these protocols were
definitely reserved for the elite in the field. Raft [1], a popular consensus library which aims to bring the
protocols to the mass, only appeared in 2014. 

I will never be able to do justice to the complexity of distributed consensus and the vast design space it
induces, and I do not plan to do so in this post. Instead, the following is merely a collection of notes I
gathered from reading relevant papers. 

Before moving on, let us get the definitions out of the way. Consensus protocol are characterized by two
properties:

+ **_Safety_**:   all non-faulty nodes agree on the same value. 
+ **_Liveness_**: the system makes progress, i.e. it does not get stuck in any round of agreement. 

They are so embedded into the design of existing large-scale systems that we may not see them being mentioned
explicitly by names. Nevertheless, they are key to two major designs in distributed systems:

+ Replication:  data and application are replicated to increase availability and to tolerate failure.
Replicas must agree on the same state for replication to be useful. The canonical example for this is
replicated state machine. 
+ Partitioning: application states may be partitioned to increase load balancing and scalability. All the
partitions must agree on the sequence of operations. The canonical example is a distributed, transactional
database system. 

### FLP
There are inextricable links between consensus and fault tolerance. In fact, when there is no failure in the
system, reaching consensus is rather straightforward. Most complexities arise from the requirement to tolerate
failure. And whenever failure is possible, we are bound by the FLP [impossibility result](../flpcap). That is,
**consensus (both safety and liveness) is not possible in asynchronous network**. But of course this was not
as negative as it sounded, since I am still here writing about this 30 years later. 

Why then, despite FLP, are consensus possible in practice? The answer is two-fold:

  1. Almost all consensus protocols choose safety over liveness, i.e. they are not live under network
  asynchrony. Some has claimed that liveness without safety is meaningless, but
  Bitcoin is an example of a _live and probabilistically safe_ protocol, and Bitcoin is far from meaningless. 

  2. To achieve liveness, existing protocols assume the network is only partially asynchronous, i.e. a message
  sent repeatedly with a finite time-out  will eventually be received. 

So FLP still holds, but in practice assumptions can be made about the network to make consensus possible.  

### Two types of consensus
Consensus protocols differ widely in the types of failure they tolerate. One is non-Byzantine (or fail-stop,
or crash) failure, in which faulty nodes simply stop responding (and later they will recover). Another is
Byzantine, in which there are no restrictions to what faulty nodes can do. The former is popular in a
distributed settings with a single owner (like a data center), whereas the later is similar to the open,
decentralized settings like P2P networks. Almost all distributed databases adopt this fail-stop model.
Byzantine failure models are often made in the context of security, particularly when nodes are compromised,
or participants in the network are malicious. 

#### Non-Byzantine 

Paxos [4] and Viewstamped Replication (VR) [5] are the first two consensus protocols. They are essentially the
same, with the latter being more easier to understand (and later implemented in Raft). 

1. The system progresses through a sequence of _views_, each view consists of a leader and a set of follower.
The leader of view $$v$$ is the one with ID $$v \ mod\ n$$. 
2. In each view:
   + Client requests are sent to the leader. 
   + Upon receiving a request, the leader broadcasts it to the followers. 
   + The leader collects a _majority_ of matching responses from the follower, replies to the client and
   commits to the request.  
3. View change is performed when either (1) leader fails, or (2) replicas time-out at processing requests. 
   + View change protocol elects a new leader. 
   + It is the most subtle part of the protocols, as commits from previous views must carry over to the new
   views.  

This is similar to two-phase commit, but instead of waiting for responses from all replicas (hence not
fault-tolerant), the leader only waits for a _majority_.  

**Why $$(2f + 1)$$ replicas**. To tolerate $$f$$ crash failures, these protocols need $$2f+1$$ replicas. This
number is optimal, for the following reason. Because there can be $$f$$ failures, there must be at least
$$f+1$$ in order for the request to be processed. If there is only 1 single agreement to be made, $$f+1$$ is
sufficient. However, we want the system to _progress_ over multiple round, that is, whatever committed at
round $$t$$ is carried over to round $$t+1$$. With this requirement, $$f+1$$ replicas is not enough, since
the non-faulty node at round $$t$$ may be faulty at round $$t+1$$, thus the operation processed at round
$$t$$ is lost. $$2f+1$$ is the minimum number that offer _**quorum intersection**_, which guarantees that
there is at least one node survive two consecutive rounds. In particular, the protocol requires responses
from $$f+1$$ replicas, say $$(0,1,..,f)$$, because then in the next round, even if $$f$$ of them fail, there
will be at least one replicas in $$[0,f]$$ present in majority quorum.  

**What happen to the remaining $$f$$ replicas**. The leader requires $$f+1$$ replicas to response, because it
assumes worst-case scenarios that the remaining $$f$$ are faulty. After these faulty nodes recover, they can
catch up with other via some kinds of synchronization.  

#### Byzantine 
The first major work considering non-crash failure models is Practical Byzantine Fault Tolerance (PBFT) [6] by
Castro and Liskov in 1999 (a decade after Paxos). The paper is surprisingly easy to read in comparison to
Lamport's original Paxos paper, perhaps because it was presented as an extension to VR. In fact, the system
also moves through a sequence of views, but there are a number of differences:

+ Authenticated messages: unlike crash failure, nodes must be authenticated, to prevent malicious parties from
creating or forging identities. 
+ In-view protocols: instead of 2 phases, there are now 3 phases of message exchange before replies can be
sent to the client. Specifically:
  1. Client sends request to the leader. 
  2. Leader broadcast PRE-PREPARE to followers. 
  3. Each follower broadcasts PREPARE to other nodes, and waits for $$2f+1$$ matching PREPARE messages
  (including its own). 
  4. Each follower then broadcasts COMMIT to other nodes, and waits for $$2f+1$$ matching COMMIT messages
  (including its own). 
  5. Each follower executes the operation, then sends REPLY to the client. The client waits for $$f+1$$
  replies before returning. 
+ View changes: initiated by replicas when they detect/suspect leader failure, the new leader must prove that
at least $$2f+1$$ replicas (including itself) wanted a new view. Like in VR, view change is the most complex
part of the protocol, and in this case it includes many large messages which carry signed proof and logs of
previous operations.  


**Why $$(3f + 1)$$ replicas**. To tolerate $$f$$ failures, at least $$3f+1$$ replicas are needed. This number
is optimal. Suppose we only have $$2f+1$$ replicas and the protocol uses majority votes of $$f+1$$. Suppose
further that nodes $$0,1,..,f-1$$ are faulty. In round $$t$$, nodes $$f+1,..,2f$$ do not receive messages
(because of temporary network partition, may be caused by the Byzantine nodes), and the operation
$$o_t$$ is committed successfully in the quorum of nodes $$0,1,..f$$.  However, in round $$t+1$$, node $$f$$
is prevented from communicating with others, and the Byzantine nodes lie to non-faulty nodes that the last
committed operation is $$o_{t'} \neq o_t$$, which violates safety property.  Thus, _more replicas are needed
to detect faulty nodes lying/equivocating_. $$3f+1$$ is the minimum size with the property that any quorum of
$$2f+1$$ replicas intersect at at least $$f+1$$, hence at least 1 non-faulty node survive to the next round,
even when the adversary can stop any arbitrary node from communicating

A more intuitive explanation for $$3f+1$$ is the following. Since the system wants to make progress, it should
wait for only $$n-f$$ responses, since $$f$$ of them may be faulty. But the Byzantine node may simply
preventing these $$f$$ nodes from communicating, thus the faulty nodes are actually inside the $$n-f$$ quorum.
To tolerate $$f$$ failures in this $$n-f$$, there must be more non-faulty nodes than faulty ones, i.e. $$n-f-f
\geq f+1 \leftrightarrow n \geq 3f+1$$. 


### Recent works in non-Byzantine consensus

### Recent works in Byzantine consensus

[1] Raft: In search of an understandable consensus algorithm. https://raft.github.io/

[2] UoW notes

[3] MIT notes

[4] Paxos

[5] Viewstamped Replication

[6] PBFT
