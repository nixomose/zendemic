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

Data never leaves your machine unencrypted. All data is encrypted, then stored on the remote 
zendemic object store node.

# Blocks sizes

Whenever you put a file or create an object, it is defined with a block size. The chunks of the object data
are stored in these blocks. The block size can be any power of two
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

Erasure coding is not supported, only this mirroring functionality. 



# Distant and local peers.

zos will work with remote nodes (other nodes on the zendemic network) that can be physically close or far away.
Normally, all traffic is routed through the zendemic network of nodes via the zendemic application.
But for zos, there is an optimization that will allow zos nodes to connect directly to each other if they are
routable. zos status will show you any active direct connections. they automatically disconnect when idle for a few minutes.


# Bundles

When creating an object and storing data in it in zos, the blocks of data themselves
are stored in 
the cloud network of zendemic object store attached peers. That is to say other people running the zendemic object store
will be storing the encrypted data that comprises your object. 
The metadata for that object however, is stored locally on your machine only. So the 
organization of the object, which block of data goes where in the object, is tied to your machine.
There is initially only one copy and it is stored only on your machine. So even if somebody somehow
had access to all of the blocks in your object (not all blocks for a given object are neccesarily stored on the same peer)
and if they were also somehow able to decrypt the blocks of data you store on their machine, they would have a hard time
piecing it together into the object you had stored.
This creates the problem that there isn't a lot of redundancy for your object's metadata, and if your 
local zendemic object store instance was destroyed, although the raw encrypted data would be safe elsewhere in
the zendemic network, you wouldn't be able to piece your objects back together without the metadata.

Enter bundles.

Any zos object can be bundled into a relatively short string, called a link, of 300 or so characters.
What bundling does, is take all the metadata for your object and bundles it up into another object.
That object is stored, redundantly, in the zendemic object store network just like any other object.
If you have a large object, and the metadata for that object ends up being bundled into an object that is
also large, it will have *its* metadata bundled, and so on until the link becomes reasonably short.
The result of bundling an object is that link string printed out to the display.
You can email that link, or copy it to a file, or take a picture of it, but that string represents
your entire object in bundle form. 
You can also unbundle a link back into an object. 

There are a few uses for bundles:

1) it enables you to render any object into a string that can be easily stored and quickly and easily copied.

2) it enables you to make a redundant copy of your object's metadata. By storing the metadata as
an object, that metadata is now stored redundantly in the zos network. You need only the link to access it.

3) it effectively creates a snapshot of your object at that point in time you bundle it.
So if you want to keep the state of an object at a particular point in time, you can bundle it
into a link, and later refer to that object at that point in time by unbundling the link.

4) Bundling makes it possible to effectively move really large objects really long distances.
A long time ago it became clear that the fastest way to move a lot of data from one side of the US
to the other is ups or fedex. Bandwidth is limited, so if you have a lot of data, it really does take a
long time to transfer it over the internet, such that it will at some point be quicker to take a pile of 
hard drives and physically ship them to your destination.
Since the actual data in an object is stored all over the zos network, you only really have to move
the metadata to get an object from one place to another, and you can bundle the metadata into a link.
So if you have a huge object, you can bundle it into a link, email that link to somebody far away
and they can unbundle the link back into an object, and then access the object, without ever having
to actually move all the data anywhere.





# Sponging





