---
layout: post
title: "Bigger NFS rsize, wsize options than 64k not possible on Netapp. Why?"
category: posts
---

We tried to do some performance tests for OpenStack environment where NFS protocol was used for backend connectivity between nodes and Netapp. One of our tests was supposed to test maximum throughput, so we tried to use maximum possible rsize and wsize options for mounts (up to 1 megabyte).

As a client we used oldish Debian system (kernel 2.6.32-5-68). We mounted test volume with 256k rsize and wsize options and expected that we will see read and write request bigger than 128k in filer statistics:

    client~# mount -t nfs -o v3,wsize=262144,rsize=262144 filer:/vol/NFS_TEST /mnt/NFS_TEST

then we did test write with *sio* utility and dd too with same result

    client~# sudo ./sio_ntap 0 100 256k 1g 20 4 /mnt/NFS_TEST/1g –direct

and as you can see from nfsstat on filer, no read/write requests bigger than 128k came to filer

    filer> nfsstat -h
    
    Read request stats (version 3)
    0-511      512-1023   1K-2047    2K-4095    4K-8191    8K-16383   16K-32767  32K-65535  64K-131071 > 131071
    0          0          0          0          0          0          0          0          0          0
    Write request stats (version 3)
    0-511      512-1023   1K-2047    2K-4095    4K-8191    8K-16383   16K-32767  32K-65535  64K-131071 > 131071
    0          0          0          0          918        1          1          3          809        0

### Why?
First thing what we suspected was old NFS client on our Debian server. We checked `/proc/mounts` and found out that even we used 256k in mounting options, mount was actually done with 64k values:

filer:/vol/NFS_TEST /mnt/NFS_TEST nfs rw,relatime,vers=3,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=10.228.167.98,mountvers=3,mountport=4046,mountproto=udp,addr=10.228.167.98 0 0

Reason is that rsize/wsize parameter which user wants to use is taken only as user's wish. During mounting process, client and servers negotiates on particular parameters. Our NFS client supports rsize/wsize up to 1 mbyte. You can find it in NFS source code in file `linux/include/linux/nfs_xdr.h` 

Constant NFS_MAX_FILE_IO_SIZE determine how big block size nfs client supports. But it is not the only one 

{% highlight c %}
/*
 * To change the maximum rsize and wsize supported by the NFS client, adjust
 * NFS_MAX_FILE_IO_SIZE.  64KB is a typical maximum, but some servers can
 * support a megabyte or more.  The default is left at 4096 bytes, which is
 * reasonable for NFS over UDP.
*/
#define NFS_MAX_FILE_IO_SIZE    (1048576U)
#define NFS_DEF_FILE_IO_SIZE    (4096U)
#define NFS_MIN_FILE_IO_SIZE    (1024U)

{% endhighlight %}

In the file `source/linux/fs/nfs/client.c` you can see what else affects rsize/wsize values during negotiation. Function `nfs_server_set_fsinfo` is responsible for it. You will find these lines of code there:

{% highlight c %}
        if (fsinfo->rtmax >= 512 && server->rsize > fsinfo->rtmax)
                server->rsize = nfs_block_size(fsinfo->rtmax, NULL);
        if (fsinfo->wtmax >= 512 && server->wsize > fsinfo->wtmax)
                server->wsize = nfs_block_size(fsinfo->wtmax, NULL); 
{% endhighlight %}

#### What does it mean?
This snippet of code means that if values (rtmax and wtmax) returned by NFS server (Netapp) are smaller than values specified during mounting by user options, then values from server will be used.

We did tcpdump and checked in Wireshark what values Netapp returns during mounting and there it was:

![tcpdump](/images/tcpdump_nfs256k.png)

There you can see that values which Netapp sends does not allow to use bigger rsize/wsize values.  
We have discussed this with Netapp and internal engineering confirmed this behavior.










