---
layout: post
title: "Naming in Distributed Systems"
date: 2014-02-16 22:47
categories:
 - Operating Systems
 - Distributed Systems

tags:
- '2014'
---

{% include toc %}


### Introduction

The scope of this post is limited to the study of naming systems for following system<br>

•    Directory Service for Wide Area<br>
•    File system or Content Management System for collaborative work<br>


Naming can be categorized into four kinds<br>
1. Host based naming<br>
2. Global naming<br>
3. User/Objet centered naming <br>
4. Attribute based naming<br>

The system we are going post falls into one of this category or hybrid of it. On the broad level, naming is associated with **users, hosts, services, files, objects and groups**. The requirements for directory for wide-area and Filesystem/CMS can be broadly categorized into following components:<br>
•    Scalability<br>
•    Availability<br>
•    Consistency<br>
•    Reliability<br>
•    Fault isolation<br>
•    Performance and Efficiency<br><br>


### Directory Service for WAN – DNS    †

DNS is the case study of directory service for wide area network. The various requirements of DS†† for WAN††† has been summarized and presented in the form of DNS as follows:

#### Granularity of Names

In DNS, domain names are represented by a **character strings** and machine-oriented **binary identifier** is called Internet Address. Domain name rarely changes than the host more down the **hierarchy** so, on the course of granularity the **less** frequent to change lead to **less** number of messages and less number of objects to deal. In DNS mechanism of name to machine address lookup, finer the level of granularity more it is prone to change hence leads to **more** number of messages and query flow in the network. So higher the hierarchy is lower the name granularity and lower the hierarchy is higher the name granularity.<br><br>


#### Caching of Names/Placement of Caches

All names are cached which the name server heard about from other name servers while handling the request of name resolution.<br>

• **Iterative query** – Name query goes to local name server where server matches query to the longest name prefix in its local cache. It caches the request and response for future reference.<br>
• **Recursive query** – Every level of name server maintains the cache, in other words, the multilevel caching, which tries to resolve the longest sequence of name query.  In recursive query look up the server caches the query request and as well as query response.<br>
• **Negative Caching** – Negative caching is used for bad names or absence of a resource record in order to answer future queries as quickly.<br><br>


#### Use of Replication

To ensure **high availability** and enhance **performance** of name service name servers are replicated, and the frequency of **replication** depends upon the frequency of its use and the degree off its **importance** in the network. For example, the root name server is highly replicated to ensure its high availability and avoiding frequent name queries.<br><br>


#### Use of Distribution

The hierarchical model of DNS distributes the job of managing the handing out of names by **distributing the responsibility** of operating name servers. Distribution is maintained in terms of different domain name servers for different top levels domains, so there is a natural separation in terms of sending particular kind of name queries to one name server and other kinds of name query to other name servers.<br>

More formally, namespace is delegated at every domain, and the whole space is partitioned into a number of the area called **zones**, which starts a domain and extends till leaf nodes, which is an individual computer, or to other domain where other zone starts.<br><br>

#### Consistency/Synchronization Requirement

DNS cache manager synchronizes the cache records when expired. For consistency, caches maintain the **time to live** for every entry.

For any update operation, the primary server of a zone is contacted. Each secondary server periodically establishes a communication connection with the primary server and gets the update.<br>

<hr style="border-top: 1.5px dotted black"/><br>


### File system/Content management system (CMS)

File System or CMS for collaborative work is a smaller environment as compared to directory service for a wide area. Hence, the priority of requirement we discussed in earlier changes. The scalability for small scale is not the top priority, but it remains a requirement for the future. However, other requirements like availability, consistency, etc. remains a significant requirements.<br>

CMS/File System for University can have Global naming, User-centric naming system, attribute type naming scheme. <br>


#### Granularity of Names

The granularity of names in CMS systems is broad in nature. Since the system is not massive and not distributed of the highest degree, large granularity works. Moreover, in a system like this more details can be accommodated into naming increasing the performance of the naming system.<br>

For instance, Tilde naming system is a relative naming system based on a collection of the small, disjoint, and hierarchical namespace. The level of is very less as compared to wide-area hence lower the granularity. <br>
In Prospero File System, the Virtual System Model implements the concept of closure, which reduces the granularity.<br><br>

#### Caching of Names/Placement of Caches

In CMS/File Systems environment caching of names provide enormous performance boost because the effect of the locality of reference or caching the alias plays a significant role. The cache can be most efficiently used for most frequent access file names, which is limited and manageable in case of small environment like the university. The cache can be managed on a centralized server or the primary name server based on system naming architecture and boosts the overall availability.<br><br>

#### Use of Replication

Replication is of required for high availability of naming service.  Replication enhances the performance in the university environment when using the global naming system and act as a load balancing to serve request quickly. It provides the fault tolerance by maintaining the replication of progressive collaborative work.<br><br>

#### Use of Distribution

Distribution in the global naming system if designed hierarchy, in the case of Prospero, local name server associated serve each request. Next component is resolved by directory server in the response of local name server. So, the naming service is distributed in the context of the processing of user name query in **distributed fashion** rather than query served by dedicated **one name server**. Maintaining a non-distributed global name service irrespective of **distributed file content** can be a **bottleneck** and global name server **performance issue**, when files across the server are moved and renamed frequently.<br><br>

#### Consistency/Synchronization Requirement

Consistency is achieved by synchronization and in a collaborative environment like university synchronization is a high priority. Stale name values in cache or time to live values in the cache can lead to poor performance in a dynamic, collaborative environment.  Although frequent synchronization leads to degrading the system performance but in order to provide consistency and reliability, it can be compromised.



<br><br><br><br>
† Domain name service<br>
†† Directory Service<br>
††† Wide Area Network<br>
