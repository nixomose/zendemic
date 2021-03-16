# Zendemic Object Store.


# Overview

The Zendemic Object Store is an object store backed by other nodes on the zendemic network
also running the zos software.

The idea is you share some space on your local disks, and other people share out some space on their
local disks, and everybody has a huge pool of disk to use to store objects or block devices.

The zos command line tool is used to interact with the zendemic object store system.

There are commands to create empty objects, put a file into an object, get a file out of an object
and attach a block device (linux only at the moment) to an object.

These commands are listed in zos.md

The list of nodes running the zos software can be large, so you can join one or more diskgroups
that define which nodes will use your disk.

All diskgroups are password protected, so you need to know the shared secret a zos node is using before
you can use that node's storage.

Zendemic Object Store diskgroups should not be confused with zendemic zones.

A zendemic zone defines a group of zendemic nodes that will communicate with each other.

Once you have zendemic nodes that can communicate together, zos nodes can find each other, but
only if those nodes are part of the same zendemic zode and are members of the same diskgroup
and those diskgroups have the same shared secret.

If all that is true, the zendemic object store nodes should find each other and you should be able
to use those nodes' storage for your objects.

# Blocks sizes

Whenever you put a file or create and object, it is defined with a block size. The block size can be any power of two
from 4k to 1 meg.
If not specified when 'zos put'ting a file, the block size will be set to the next largest valid block size that will contain the whole
file. If the file is larger than 1 meg, the block size will be one meg.

When creating an object to be used for a block device, you must specify the block size, same rules, power of two
between 4k and 1 meg.

With a block device though, there is more to consider than just the size of the data you'll be writing.
All requests from the block device come into the system as 4k requests. If there are a bunch of adjacent 4k requests
they will get grouped together into one request.

However if a write request doesn't contain all the 4k segments in a zos object block, the block device handler will have to
read the block, update the 4k segments being updated and write the block back out.

For example, if you make the object block size 1 meg, and the block device user writes 4k to some part of it, zos
will read the 1 meg block that contains that 4k from one of the remote nodes holding the data, 
update the 4k and then write out all the copies of the new block.
This creates a lot of overhead if there's lots of random 4k io.

On the other end of the scale, if you make the object block size 4k, you will never have to do a read-update-write operation,
but you will cause lots of unneccesary overhead for lots of adjacent writes. Writing a 1 meg chunk aligned to the block size will
cause 256 writes where only one write would need to be done if the block size was 1 meg.

So size your block size appropriately for your workload, run tests with different block sizes to see what works best for 
your workload.





# Block copies 

By default when creating an object, zos will define the object specifying that it write three copies of every block.
No two copies of the same block will land on the same node.
You can specify from 1 to 7 copies of every block. All blocks are written in parallel (assuming no failures) 
so there isn't much of a performance penalty for specifying more copies, but consider how much disk you'll be using
compared to what you actually need.

Erasure coding is not supported, only this mirror-like functionality. 



# Distant and local peers.

zos will work with remote nodes (other nodes on the zendemic network) that can be physically close or far away.
Normally, all traffic is routed through the zendemic network of nodes via the zendemic application.
But for zos, there is an optimization that will allow zos nodes to connect directly to each other if they are
routable. zos status will show you any active direct connections. they automatically disconnect when idle for a few minutes.






