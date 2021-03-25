# Project 4: Finishing Our Router Implementation

This project is to be done individually.  Please refer to Canvas for the deadline.

## Introduction

In HW2, you implemented a simple 'static' router.
The router could act as a layer-2 endpoint, demultiplex to layer-3, forward IPv4 packets, and handle ARP requests.
However, all of those behaviors were statically configured in the control plane.
In HW3, we saw how routers actually implement dynamic behavior like the distributed computation of routing tables.

In this project, you will combine what you did in HW2 and HW3 to fill out your Internet router implementation.
Specifically, you will import your implementation of the distance vector protocol to our P4/Python-based network.
To help make sure that your HW2 and HW3 implementations are solid, we will provide additional access to grading scripts and include as part of the timeline some buffer that you should use to address any open issues in your implementations.


## Part A: Cloning the Updated Framework

You can clone and build the framework exactly as you did in HW0 - HW2.
Note that this repository includes some updates the the framework that will help you in your implementation.
We will discuss these updates below.

For now, the topology we will use in this homework is identical to HW2:

![Topology](configs/topology.png)


Eventually, you will want to start copying in your HW2 `data_plane.p4` and `control_plane.py` code into this repository.
Before you do that, however, note some changes that we have made to the framework.
The purpose of these changes is to support bi-drectional packet communication between the data and control plane.
Think about the types of packets in your HW3 implementation.
*Traceroute packets* were handled entirely with your routing/forwarding tables; these correspond in real networks to normal IP packets.
*Routing packets* required a bit more computation; in real networks, these are sent up to the control plane and the control plane will send routing updates back down to the data plane.
Digests can be used to ship information from the data plane to the control plane, but thus far, we have not seen a mechanism to ship packets back to the data plane -- only commands to configure it.

### Sending packets to the data plane

`switch.py` includes a new function `SendPacketOut(payload)` that can be used to send packets to the data plane.
The payload parameter to this function should be a sequence of bytes represented as a Python string.
The data plane receives this payload as a raw frame from ingress port 255 (`CPU_PORT`).
The term raw frame indicates that the switch can tell that it's a single message, but there will be no additional encapsulation on the packet.
Thus, if you wanted to send an IPv4 packet to the data plane, you would need to generate the headers manually before calling this function.

### Data plane changes

To help you out, we have defined a simple distance vector packet format in `data_plane.p4`.
The format assumes a maximum of 10 IPv4 destinations.
It is parsed in the parser's `parse_routing` state.
You can deduce the valid header combinations involving this format from the parser code.
Specifically, take a look at (1) the select statement of the parser's `start` state, and (2) the select statement of the `parse_ethernet` state.
You should be able to answer these questions on your own:

>For frames coming from the local control plane, what do the frames contain?  Do they have any headers?

>For frames coming from other routers in the network, what is the sequence of headers?


### Control plane changes

For convenience, we have provided a function called `buildRoutingPayload(my_ip, distance_vector)` in `helper.py`.
The function takes an array of [destination, cost] tuples and packs them into a payload that can be sent to the data plane using the `SendPacketOut` function.
It also takes the current IP and includes it in the distance vector routing packet.

We have also provided some skeleton code in `control_plane.py` that demonstrates how to send packets and receive digests simultaneously.
The key extra functionality here is the separate update thread.
We need to spin up a separate thread in this case as reading digests is a blocking operation that could potentially prevent updates from ever being sent out.
Note that multiple threads within the same control plane potentially introduces concurrency issues -- you should use locks to handle this.
We have provided a simple Python threading lock `dvLock` that you can use for this purpose.


## Part B: Finishing your router

Your goal in this project is to take your code from HW2 and remove all static routing table entries.
You are still allowed to preconfigure other aspects of your router.
Specifically, you can include in the control plane knowledge of:

  - the number of ports per switch
  - the local IP and MAC of each port
  - the set of static ARP table entries for each port

Nothing else should be specific to our topology.

We will allow you to make two simplifications to the distance vector protocol.
The first simplification is that all links have cost 1.
In effect, our definition of "shortest path" in this network only depends on hop count.
The second simplification is that the only destinations in the distance vector are hosts -- routers should *not* be in the distance vector.
Next hop addresses should still be correct in your routing tables and packets.

To assist in integrating hosts into your protocol, we have configured them to send out routing updates every 10 seconds.
You can look at `utils/send_host_routing.py` to see the script that does this.  


**Tip #1**: This is the first project in which we are dealing with threading.
You should take care as there is a large class of bugs that can arise due to concurrency issues.
Common mistakes include not locking access to shared objects.
They also include misconceptions about Python parameter passing semantics.
Python uses pass-by-reference which means that modifications can be observed in both the control plane thread and update thread.
Reassigning the reference will break this connection, e.g., `distanceVector = []` in the update thread.

**Tip #2**: Again, we recommend that you break the functionality into pieces.
For instance, you should be able to start by ignoring the update thread and focusing on the getting the distance vector from the data plane to the control plane.


## Submission and Grading

Submit your `data_plane.p4` and `control_plane.py` to Canvas.  Make sure to **Indicate your name and PennKey in a comment at the top of both files.**

As always, start early and feel free to ask questions on Piazza and in office hours.


