---
layout: post
title:  No-stale-reads vs real-time ordering
date:   2022-03-01
description: Some discussions on consistency models. 
tags: consistency distributed-database
categories: thoughts
---


TL;DR

For single key, no-stale-reads considers each read independently, 
while linearizability also restricts each read value w.r.t. other reads.
For multiple keys, things become a little bit complicated the overall theme is the same.

## Background

When doing my projects, many times I got confused with the guarantee
of "the system provides no stale reads" (i.e., regularity) and "the system preserves
all real-time ordering requirements" (i.e.,  linearizability). I found two constructions to distinguish
them. The first construction (from 
{% cite nonstale:stack --file ref %} is for single-key operations as shown below, where
`wwww` and `rrr` means the duration of one single write/read operation. 
```
Key(x);  0                  1
P1:         wwwwwwwww(x=1)
P2:                rr(0)        
P3:         rr
```
In this construction,
P3's read may return 0 or 1 if the system ensures "no stale reads"
because the read is concurrent to the write. However, P3's read must return
0 as it precedes P2's read in real-time: P3's read must always be ordered before 
P2's read in any history/schedule. Symmetrically, if P3's read returns 
1, then P2's read must not return 0.



The second construction is for transactional systems (the construction in my DAST paper {% cite chen2021achieving --file ref %}).
Consider the following construction 
with two keys A and B. T2 and T3 only access one key each, while T1 is a distributed 
transaction accessing both keys. 
In this construction, we have $T2 -> T3$ in real-time. However, the serializable order
(i.e., the equivalent serial schedule for serializability)
among these three transactions is $T3->T1->T2$, which breaks the real time-order. 
This construction does ensure no stale reads, but does not preserve all real-time 
ordering requirements. 
```
A:   T1    T2
B:              T3    T1
```

Recently, I read the RSS paper {% cite helt2021regular --file ref %}
, which helps to shed light on this problem. 
RSS ensures all reads see finished writes (i.e., no stale reads) and preserves 
the real-time ordering from write operations/transactions to other operations. 
RSS is proved to be *invariant-equivalent* to linearizability.
To understand the idea of invariant-equivalence, we need to first distinguish the
definitions of *invariants* and *anomalies*. Usually, invariants are defined 
from system/service's pov, while anomalies are defined from the external view. 
For instance, a system/service cannot track whether two processes/clients 
contacts externally (e.g., Alice calls Bob after its operation finishes). Therefore, 
we usually refer to strict serializability and linearizability as *external consistency*:
the system/service is consistent even if one views the external world altogether where
processes/clients can communicate with no delay.



RSS paper writes, "like 
all prior *regular* models, (RSS ensures that) reads must return a value at least 
as recent as the most recently complete conflicting writes". 
This sentence motivates me to dig deeper into the classical papers about
consistency levels. 

## Shared Registers

The basic single-write multiple-writer shared register model was first proposed by 
Lamport in 1986 {% cite lamport1986interprocess --file ref %}, together with the *atomic, regular, and safe* consistency models. 
This model only has one writer and one key, which serve as the base for all 
subsequent consistency models. 

- A safe register ensures that a read not overlapping with writes returns the latest 
writes; otherwise, the reads can return any value (even a corrupted one). 
Note that in the basic model, 
there is only one writer, so no concurrent writes. Such registers may show in shared 
registers on chips {% cite lamport1986interprocess --file ref %}.

- A regular register is a little stronger. It also requires that reads must return the 
value that is at least as recent as latest finished writes, while it can freely 
choose whether to return the values of concurrent writes. 

- An atomic register provides the strongest guarantee. In addition to previous requirements, 
it also requires that a subsequent read cannot returns a earlier value than a read 
finished before it (i.e., linearizability).

## Back to the motivation cases.


It is obvious that in the first construction, a consistency model ensuring this can be called as 
*regular*, like in RSS. Note that this is not Monotonic Read guarantee, as MR guarantee is defined
only within session. It is also *sequential*, but not linearizability. 

In the second construction, however, things are a little complicated as the original model 
only involves one register. 
In essence, the difference between the two constructions is that P3's read is read-only, while T2 is not. 

The key ambiguity sources from the fact that Lamport's original paper only defines 
regularity for single register: read must see a version at least as recent as 
the latest finished writes. However, this construction has two registers, but the definition 
does not discuss the relation of two (unrelated) writes on two different registers. 
As far as I am concerned, there 
is only one work {% cite viotti2016consistency --file ref %} that
defines regularity for multiple keys with the following three requirements: 

* There is an equivalent total order (a.k.a., the arbitration order) of all requests.  
* This order respects the real-time order (a.k.a., the "return before" relation) from writes to any operations. 
* The system is return value consistent, i.e., the return value is as expected. This is 
      a very weak guarantee, but the atomic register model does not ensure this. 


This definition also requires the real-time ordering from a write to an un-related 
operation must also be preserved. The RSS paper follows this definition. If we adopt
this definition, the second construction is *not regular*, as it does not 
the real-time ordering between T2 and T3 if T2 contains write operations. 



However, this general definition has not reached an agreement in academia/industry,
as the intuition behind Lamport's definition of "regularity" is that the register 
has no stale reads: all finished writes must be visible to subsequent reads
until it is overwritten. 


I will discuss more regular consistency models defined in {% cite shao2011multiwriter --file ref %}
on multi-writer regularity
in subsequent posts. 

References:
----------

{% bibliography --file ref --cited --template post-ref %}



