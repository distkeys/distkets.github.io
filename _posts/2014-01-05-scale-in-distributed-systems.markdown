---
layout: post
title: "Scale in Distributed Systems"
date: 2014-01-05 09:41

categories:
- Distributed Systems
- Operating Systems

tags:
- '2014'
---

{% include toc %}

### Introduction

This blog is summary of the research paper [Scale in Distributed Systems.](https://1drv.ms/b/s!AqRlTEkVrXvJg6RV8zaLTZs26ymmCg)


A system is Scalable if it can handle addition of users and resources without suffering a noticeable loss of performance or increase in administrative complexity.

&nbsp;

### Systems designed to Scale

Some historic systems desgined to scale

* <a href="http://courses.cs.washington.edu/courses/cse451/07au/lectures/pending/p260-birrell.pdf" target="_blank">Grapevine</a>, <a href="http://dl.acm.org/citation.cfm?id=2081&dl=ACM&coll=DL&CFID=395866995&CFTOKEN=19566412" target="_blank">ACM Paper</a>
* <a href="http://en.wikipedia.org/wiki/Domain_Name_System" target="_blank">DNS</a>
* <a href="http://en.wikipedia.org/wiki/Kerberos_(protocol)" target="_blank">Kerberos</a>
* <a href="http://en.wikipedia.org/wiki/Sprite_(operating_system)" target="_blank">Sprite</a>
* <a href="http://research.microsoft.com/en-us/um/people/blampson/36-GlobalNames/WebPage.html" target="_blank">Global name services and authentication services</a>
* <a href="http://en.wikipedia.org/wiki/LOCUS_(operating_system)" target="_blank">Locus</a>
* <a href="http://en.wikipedia.org/wiki/Andrew_File_System" target="_blank">Andrew</a>
* <a href="http://en.wikipedia.org/wiki/Project_Athena" target="_blank">Project Athena</a>
* <a href="http://www.cdf.toronto.edu/~csc469h/fall/handouts/nitzberg91.pdf" target="_blank">Dash</a>
* <a href="http://en.wikipedia.org/wiki/Amoeba_(operating_system)" target="_blank">Amoeba</a>


As systems scales and accessible objects grows, locating objects of interest becomes difficult. Systems addressed this aspect of challenges are

* <a href="http://en.wikipedia.org/wiki/9P" target="_blank">Plan 9</a>
* <a href="" target="_blank">Profile</a>
* <a href="http://gost.isi.edu/products/prm/papers/prm-cpe.ps" target="_blank">Prospero</a>
* <a href="http://ieeexplore.ieee.org/xpl/articleDetails.jsp?arnumber=4798" target="_blank">QuickSilver</a>
* <a href="https://www.usenix.org/legacy/publications/compsystems/1990/fall_comer.pdf" target="_blank">Tilde</a>

&nbsp;
Paper talks about problem of scale and general solution. These solution fall into three categories


> Replication, Distribution and Caching


<br><br>
![alt text](/images/Scale.png)

<br><br><br><br>

### Naming and Directory Services

A name refers to an object. An address tells where that object can be found. The binding of a name is the object to which it refers.


> A name server (or directory server) maps a name to information about the name's binding.



#### Granularity of Naming


> The granularity of the objects named affects the size of naming the database, the frequency of queries and the read-to-write ratio which affects techniques can be used in naming in large systems.


In an approach of fine grained naming, files must include name of the host so that object could be located.


> <mark>Problem</mark> with finer grained naming is that moving objects is difficult since objects are tied to the server in their names.

 Another naming approach is common prefix naming, where group of objects share a common prefix and name service maps the prefix to the server. The remainder of the name is resolved locally by the server on which the object is stored.

**Advantages**
 <br>
 1. Clients can cache the mapping keys as prefix changes less frequently than the rest of the name<br>
 2. Since objects does not have to include server name, it is easy to move objects around

**Disadvantage**<br>
 Objects sharing common prefixes should be stored together
<br><br>

#### Reducing Load

* Replication - When multiple name server handle the same queries, different clients are able to send their requests to different servers.

**Choice of Server**<br>
1. Physical Locations<br>
2.  Relative loads on servers<br>
3. Random server selection<br>

<mark>Problem</mark>
Its hard to make replica consistent.

* Distribution - Spread the name resolution load across servers.

In distribution, namespace is assigned to different servers.


**Advantages**<br>
1.  Since part of the naming database is stored on each server, thus reducing number of queries and updates to be processed.<br>
2.  As size of each database is reduced, each request can be handled faster.<br>
3.  The client must be able to determine which server contains the requested information.
<br>

* Caching<br>
By Caching the mapping from a prefix to the name server handling that prefix, future names sharing the same prefix can be resolved with fewer messages.

<mark>Problem</mark>

> Caching is a form of replication and as similar to replication biggest difficulty is the need to for consistency.


#### Unique Identifier based naming (UID)

UID based naming is based on UID which is used to name and grant access right to the object.

UID may be thought as addresses. It contain server information identifying the server address and an identifier to be interpreted by the server to locate the object.

<mark>Problem</mark>
<br>

UID identify the server on which an object resides. When object moves to other location the UID is no longer valid for the object they reference.<br>

This problem can be solved using technique called forward pointers. With forward pointers a user attempting to use an old address to access an object is given a new UID containing new address.

<mark>Problem</mark> with forward chaining is that the chain of links can become lengthy and reduce performance. Also, if one of the node is down then access to link is prevented.

This problem can be solved by requiring each object to have a home site with the forward pointer at that site kept up-to-date.

<a href="http://gost.isi.edu/products/prm/papers/prm-cpe.ps" target="_blank">Prospero</a> supports UIDs with expiration dates. Its directory service guarantees that the UIDs it maintains are kept up-to-date. The use of expiration dates makes getting rid of forwarding pointers possible, once all possible UIDs with the old address have expired.
<br><br>

#### Directory services

Directory servers translate symbolic names into the UIDs. A directory can contain UIDs for files, other directories, or- in fact- any object for which a UID exists. The load on directory servers is easily distributed as different parts of a name space can reside on different machines.

The primary difference between a name server and a directory server is that the directory server usually possesses little information about the full name of an object. A directory server can support pieces of independent name spaces.

Prospero and Amoeba use directory servers to translate names to UIDs
<br><br>

#### Growth and reorganization

When two organizations with separate global name spaces merge, reorganize, or otherwise combine their name spaces, a problem arises if the name spaces are not disjoint.

New names and new prefix is assigned with old name and it is a problem for any names that were embedded in programs or otherwise specified before change.

DEC's <a href="http://research.microsoft.com/en-us/um/people/blampson/36-GlobalNames/WebPage.html" target="_blank">global name service</a> addresses the above problem<br>
1. Associating a unique number with the root of every independent name space.<br>
2.  When a file name is stored, the number for the root of the name space can be stored along with the name. <br>
3. When name spaces are merged, an entry is made in the new root, pairing the UID of each previous root with the prefix required to find it.<br>
4.  When a name with an associated root UID is resolved, the UID is checked; if it does not match that for the current root, the corresponding prefix is prepended, allowing the embedded name to work.

<hr style="border-top: 1.5px dotted black"/><br>
<br>


### Security Subsystem

As system size grows, the number of points from which an intruder can enter the network increases and security becomes increasingly important and difficult to implement.

#### Authentication

*  Password based authentication<br>
This technique involves using password on each host.

<mark>Problems</mark>
1. Requires maintenance of a password database on multiple nodes.<br>
2. Its cumbersome, if user have to present password each time a new service is requested.<br>
3. Letting the workstation remember the user's password is risky.<br>
4. Vulnerable to the theft of password by eavesdrop on the network.<br>

*  Host based authentication<br>
In this technique, client is authenticated by the local host. Remote servers trust the host to properly identify the client.

*  Encryption based authentication<br>
In this technique, passwords are never sent across the network instead each user is assigned an encryption key. The key is used to prove the user's identity.
<a href="http://en.wikipedia.org/wiki/Kerberos_(protocol)" target="_blank">Kerberos authentication protocol</a> is example of large scale system authentication.
<br>

#### Authorization

**Approach 1**<br>
Client send authorization request to server and server forwards request to authorization service.

Disadvantage of this approach is access control service can become bottleneck.

**Approach 2**<br>
Client is authenticated and then server makes its own decision about whether or not the client is authorized to perform an operation.

In <a href="http://en.wikipedia.org/wiki/Andrew_File_System" target="_blank">Andrew File System</a>, each directory has an access control list (ACL) that identifies the users authorized to access the files within the directory. ACL entries in Andrew can contain the names of groups and user can be part of one or more groups.


> A disadvantage ACL model is that the client must first be authenticated, then looked up in a potentially long list; the lookup may involve the recursive expansion of multiple groups and may require interaction with other servers.


Another authorization model is capability based authorization in which the user maintains the list of the objects for which access is authorized. A client presents its capability when it wishes to access an object. The server then compares the bit pattern of the capability with that stored along with the object; if they match, the access is allowed.

**Advantages**<br>
1. The server can make its access control decision without contacting other servers.<br>
2. The server does not need to maintain a large authorization database that would be difficult to keep up-to-date in a large system.<br>

**Disadvantages**<br>
1. Capabilities can only be revoked in a group.<br>
2. Capabilities are revoked by changing the bit pattern, but this causes all outstanding capabilities for that object to be immediately invalidated. The new capability must then be reissued to all legitimate users.<br>
<br><br>

#### Accounting

Most distributed systems handle accounting on a host-by-host basis. A distributed, secure, and scalable accounting mechanism is needed, especially in large systems that cross administrative boundaries.<br>

1. Bank server approach - Amoeba uses this approach where bank servers handle accounting by maintaining accounts on behalf of users and servers; users transfer money to servers, which then draw upon the balance as resources are used.<br>
2.  Proxy-based accounting - proxy-based accounting is tied
much closer to authentication and authorization. The client grants the server a proxy that allows the server to transfer funds from the client's account.
<br>

**Replication, Distribution and Caching in Security**<br>
When these techniques are applied in the area of security, the following considerations must be kept in mind:<br>

1. When a server that maintains secret keys is replicated, the compromise of any replica can result in the compromise of important keys.<br>
2. The security of the service is that of the weakest of all replicas.<br>
3.  When distribution is used, multiple servers may be involved in a particular exchange.<br>
4. It is important that both principals know which servers were involved so that they can correctly decide how much trust to place in the results. <br>
5. The longer credentials are allowed to be cached, the longer it will take to recover when a key is compromised.

<hr style="border-top: 1.5px dotted black"/><br>
<br>

### Remote resources

So far, we saw effect of scale in naming and security system. One cannot access a resource without first finding it. This involves both identifying the resource that is needed and determining its location, given its name. Once a resource has been found, authentication and authorization might be required
 for its use.<br>

 The services used to access remote resources are very dependent on the underlying communications mechanisms they employ. This section looks at the scaling issues related to network communication in such services.
<br><br>

#### Communication

As the medium of communication places limits on the system's performance, it can greatly affect the usability of a system. The underlying communications
parameters must not be completely hidden from the application.


> In <a href="http://www.cdf.toronto.edu/~csc469h/fall/handouts/nitzberg91.pdf" target="_blank">The Dash System</a>, when a connection is established, the application can require that the connection meet certain requirements; if the requirements are not met, an error is returned.


When one set of required communication parameters cannot be met (generally in low-latency connection), the application still might be able to access the resource via an alternate mechanism  {" Whole-file caching instead of remote reads and writes "}

<br>

**Communication forms**<br>
•  Point-to-point communication - The client sends messages to the particular server that can satisfy the request If the contacted server cannot satisfy the request, it might respond with the identity of a server that can.<br>
•  Broadcast communication - In broadcast communication, the client sends the message to everyone, and only those servers that can satisfy the request respond. <br>
     **Advantages**<br>
     Finding a server that can handle a request is easy.<br>
     **Disadvantages**<br>
     •  Broadcast communication does not scale well. <br>
     •  Processing is required by all servers, whether or not they can handle a request.<br>
     •  As the total number of requests grows, the load due to preliminary processing on each server also grows.

•  Multicast communication - In multicast, a single message can be sent to a group of servers. For multicast to scale, the groups to which messages are sent should be kept small.

<hr style="border-top: 1.5px dotted black"/><br>
<br>


### Replication

Replication is used to reduce the load on the server and improve the reliability and availability of the service as whole.


> Issue is the Placement of the replicas and replica consistency

#### Placement of replicas

Placement of replicas depends on the purpose of replication

Scattered Replicas<br>
• Reduces network delays or avoid network partitions.<br>
• Improves availability of the service.


Near located Replicas<br>
• Used when majority of users are local.<br>
• Improves reliability of the service.<br>
• To spread load across multiple server then replicas may be placed near one another.
<br><br>

#### Replica Consistency

At a particular point in time, a set of replicas is said to be consistent if the value of the object is the same for all readers.<br>
Taking these steps guarantees that the set of replicas read will intersect with the set written during the most recent update

• Replication of read-only information<br>
    Replication read-only information like binaries which can not be changed by normal users.<br>
    [Andrew, Athena]<br><br>
• Replication of immutable information<br>
    Changes to files are made by creating new files and then changing the
directory so that the new version of the file will be found.<br>
    [Amoeba]<br><br>
• Update all replicas<br>
    This approach allows update but require updates to be send to all replicas. <br>

<br>

<mark>Limitations</mark><br>

     1. Can update only when all the replicas are available thus reducing the availability of the system for write operations.
     2. For data consistency, absolute ordering on updates are required.
     3. Client might fail during an update.

• Primary-site replication<br>
    All updates are directed to the primary replica which then forwards updates to the other replicas.  Updates may be forwarded individually or whole database may be periodically downloaded by the replicas [Kerberos].

**Advantage**<br>

    The ordering of updates is determined by the order in which they are received at the primary site and that updates require only the availability of the primary site.

**Disadvantage**<br>

   1. The availability of updates still depends on a single server, although some systems select a new primary site if the existing primary goes down.
   2. If changes are distributed periodically, the updates are delayed until the next update cycle.

• Loose consistency<br>
With loose consistency, replicas are guaranteed to eventually contain identical data.

1. Updates are allowed even when the network is partitioned or servers are down.<br>
2. Updates are sent to any replica, and that replica forwards the update to the other replicas as they become available.<br>
3. If conflicting updates are received by different replicas in different orders, time stamps indicate the order in which they are to be applied.<br>

> With loose consistency, there is no guarantee that a query will return the most recent data.


• Quorum consensus<br>
In Quorum consensus
1. Assigning votes to each replica<br>
2. Selecting two numbers-a read-quorum and a write-quorum-such that the read-quorum plus the write-quorum exceeds the total number of votes <br>
3. Requiring that reads and writes be directed to a sufficient number of replicas to collect enough votes to satisfy the quorum.<br>
Taking these steps guarantees that the set of replicas read will intersect with the set written during the most recent update. Time stamps or version numbers stored with each replica allow the client to determine which data are most recent.

<hr style="border-top: 1.5px dotted black"/><br>
<br>


### Distribution

Distribution allows the information maintained by a distributed service to be spread across multiple servers. Distribution reduces the number of requests to be handled by each server, allows administration of parts of a service to be assigned to different individuals.

Some of the issues of importance in distribution are the placement of the servers and the mechanisms by which the client finds the server with the desired information.
<br><br>

#### Placement of servers

Information should be distributed to servers that are near the users that will most frequently access the information.

• Reduce network traffic. <br>
• Improves reliability, as it less likely that a network partition will make a
local server inaccessible.<br>
• It is desirable to avoid the need to contact a name server across
the country in order to find a resource in the next room.


By assigning information to servers along administrative lines, an organization can avoid dependence on other organizations.<br><br>


#### Finding the right server

To be continued...



### Other topics in distributed systems

• Design and practice of distributed algorithms and data structures<br>
• Analysis of the behaviour of distributed systems and algorithms<br>
• Distributed operating systems<br>
• Parallel processing on distributed systems<br>
• Resource and service discovery<br>
• Resource sharing in distributed systems<br>
• Distributed fault tolerance<br>
• Security in distributed systems<br>
• Scalability, concurrency and performance of distributed systems<br>
• Transactional memory<br>
• Middleware for parallel computations<br>
• Web services<br>
• Self-organised and self-adjusting distributed systems<br>
• Collaborative computing<br>
• Modelling distributed environments<br>
• Communication protocols<br>

