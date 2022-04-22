# zos

zos is the command line interface for the zendemic object store server.


Global options: 

`   -q quiet output, don't display activity`

`   -t timeout in seconds to wait for zos server response, default is 60`

`   -h help specific to a command`
---
---

**object management**
---
` zos createobject <diskgroup> <objectid> <password> <copies> <blocksize> <object size in bytes> `

The create object command creates a new object with the specified copies, block size and object size
definition. Copies can be between 1 and 7 depending on how much redundancy you want,
more copies will affect write performance. Block size must be a power of 2 between 1024
and 1048576. size is the number of bytes in the object and thus the block device. It must be
a multiple of the block size.

For example: zos -d createobject dg1 bdimage bdimagepw 3 4096 10485760

That will create a 10 megabyte object. As in 1024 * 1024 * 10. Like I said, megabyte.

---
`zos destroyobject <diskgroup> <objectid> <password> <blocksize> <object size in bytes>`

The destroy object command will remove the object from the object store. Once the object is
destroyed all the data is deleted or at least inaccessible. You must pass the block size
and object size to make sure you are destroying what you think you are destroying.

For example: zos destroyobject dg1 smalldisk smalldiskpw 4096 10485760

---
---
**object information**
---


`zos objectinfosimple <diskgroup> <objectid>`

The objectinfosimple command retrieves metadata information about the specified object id.


---
`zos objectinfodetailed <diskgroup> <objectid>`

The objectinfodetailed command retrieves metadata information and details 
about the object's storage location for the specified object id. If the object is 
large, this process of data gathering may take a while.

---
---
**file functions**
---
`zos put [options] <diskgroup> <filename> <objectid> <password>`

        [-b N ] block size to write the file out in, default is 4k
        [-c N ] how many copies to write, default is 3
        [-r N ] position from source file to read from, default is start of file (not implemented)
        [-w N ] position in object to write to, default is zero (not implemented)
        [-l N ] how much data in bytes to copy, default is to the end of the file. (not implemented)

The put command enables you to place a section or an entire file in the object store.

You can copy one byte or the entire file to any position in the object.

You must specify a diskgroup to store the file in, and the file is tied to that diskgroup.

You can write over sections of the object or the whole thing. Once an object is created,
the block size and diskgroup any future writes use must match the original block size and diskgroup.

Object ids are installation-unique, you can not have two different objects in different diskgroups
in the same zendemic object store installation with the same object id.

---
`zos get [options] <diskgroup> <filename> <objectid> <password>`

        [-r N ] position in object to read from, default is zero (not implemented)
        [-w N ] position in file to write to, default is start of file (not implemented)
        [-l N ] how much data in bytes to copy, default is to the end of the object. (not implemented)

The get command will retrieve a section or an entire file from the object store.

---
`zos unmark <objectid> <start> <length>`

      not implemented

unmark will effectively zero out a section of the object.

Where an entire block would be set to zeros, the block will be removed from the object store
saving space.

---
`zos truncate <objectid> <length>`

      not implemented

truncate allows you to set the length of the object in the object store.

if you make it shorter than its current length, data after the new length will be lost.

if you make it longer, the object will be padded with zeroes.

---
---
**block device**
---
`zos createbd <diskgroup> <objectid> <password> <devicename> <device timeout in seconds>`

The create block device command will create a block device in /dev with the specified device
name, and attach it to the specified objectid. The size of the block device is determined
by the definition of the object id when it was created. The command will not return
until destroybd is called for the created device.

For example: zos createblockdevice dg1 smalldisk smalldiskpw zosbd1 30

This will create a block device /dev/zosbd1 backed by the smalldisk object.

---
`zos attachbd <diskgroup> <objectid> <password> <devicename>`

The attach block device command functions similarly to the create block device command
except that it does not create the named block device, but just attaches to the existing
one. This is for the case when the userspace program is ended but the block device
still exists and you want to keep using it. This command will also not return until the
block device is destroyed.

For example: zos attachbd dg1 smalldisk smalldiskpw zosbd1



---
`zos destroybd <devicename | handle id | all-devices>`

The destroy block device command will flush all pending writes to the backing object and
remove the block device from the /dev directory. this will also cause the userspace process
servicing the block device requests to exit cleanly.

For example: zos destroyblockdevice zosbd1


---
---
**object bundles**
---
`zos bundle <diskgroup> <objectid> <bundleimagename`

The bundle command batches up the metadata defining an object into a reasonably sized link.
This link can be used to unbundle the image somewhere else, or to reproduce the image
as a copy locally, of the state of the object at the time the bundle was made.

For example: zos bundle image1 bundleimagename1


---
`zos unbundle <diskgroup> <link> <bundleid>`

The unbundle command undoes a bundle converting the link back into a complete image
the way it was at the time the bundle was created. The unbundle command can be run in
the same location the bundle was made or a different one, thus making a copy
in the same location or a different one, without having to actually copy all of the data.

For example: zos unbundle <link> newimage1copy


---
`zos destroybundle <diskgroup> <bundleid>`

The destroybundle command destroys a bundle set of images from a failed bundle or unbundle.
Bundling and unbundling create temporary objects that may not get cleaned up if the function
fails. Destroy bundle allows you to clean up the multiple temporary objects with one command.
If the bundle name is xyz, temporary objects called xyz-1, xyz-2 and so on will be created.

zos destroybundle dg1 xyz  will destroy those temporary objects.


---
---
**sponging data**
---
`zos spongeobjectstart <objectid>`


The sponge object command verifies all the data in an object. It does this by fetching at least the number
of copies specified in <copies> to make sure they are available. If not, it will replicate the data to
ensure there are the specified number of copies. YOU MUST SPONGE YOUR DATA AT LEAST ONCE EVERY 3 MONTHS OR IT WILL
BE ERASED. This is how the zos system does housekeeping. optionally selecting all-objects will start a sponge object
on all objects in all diskgroups. This happens by default on a schedule, so you should never have to manually
run a sponge, unless the zendemic object daemon hasn't been running in a while.

For example: zos spongeobject dg1 smalldisk


---
`zos spongeobjectstop`

The sponge object stop command, will stop any currently running object sponge, whether it is a single object or
if all objects are being sponged, whether it was fired off by a zos command or from the scheduler.

For example: zos spongeobjectstop


---
`zos spongestoragestart`

The sponge storage command goes through all of the data stored remotely for other node's objects and cleans
up any data that hasn't been accessed in a while (3 months). This is why you must sponge your objects occasionally.

Sponge storage will remove any data that is invalid, or hasn't been accessed.
There is no separation of data by disgroup, so sponge storage will survey all remote object's data that is stored locally.

Sponge storage runs on a schedule by the zendemic object store daemon so you should never have to manually start one,
unless you want to try and free up disk more quickly.

For example: zos spongestoragestart

---
`zos spongestoragestop`


The sponge storage stop command will stop the sponge storage process if it is running.

For example: zos spongestoragestop


---
`zos spongestatus`

The sponge status command will display any current sponge processes that are running.

    not implemented, this information is available from the zos status command.

---
---
**disk groups**
---
`zos listgroups`

The listgroups command displays all the diskgroups the local zos server participates in.


---
`zos listobjects <groupname>`

The listobjects command lists all the objects stored in the specified group name.

---
`zos addgroup <groupname> <password> <passwordhint>`

The addgroup command will add a groupname to the list of diskgroups that your zendemic object store instance
participates in. when you add a group, the data for the objects you store in the group will  be placed in the
backing storage of other zendemic object store peers who have also added that group name. 

Likewise, other peers storing objects in this group name will use your backing storage to store their object data.

The passwords must match between peers. If the password does not match between peers of the same
group name, the data will not be stored between those peers.


---
`zos removegroup <groupname>`

the removegroup command will remove the specified group name from the list of groups your zendemic object store 
participates in.
the objects in this group will no longer be available, but they are not deleted, they are hidden, and if you want
them back, you can simply add the group back.

If you do not add them back, eventually the sponging process will destroy the data.


---
---
**miscellaneous**
---
`zos listpeers <groupname>`

the listpeers command will list all of the cached peers for the named diskgroup and their available storage.

---
`zos refresh`

the refresh command triggers the zendemic object store daemon to search for peers in all of your diskgroups.

---
`zos status`

Display zos server version and other information about the zosserver.

For example: zos status
