---
layout: post
title: "Open System Call"
date: 2014-08-04 19:10
categories:
 - Operating Systems

tags:
- '2014'
---
This post will explore the code flow when Open system call is issued.

{% include toc %}

## Open System Call

For most file systems, a program initializes access to a file in a filesystem using the open system call. This allocates resources associated to the file (the file descriptor), and returns a handle that the process will use to refer to that file. In some cases the open is performed by the first access.


<br>

## Bird Eye View

Open system call goes through lot of twist and turns and explore some of the most complicated code paths. On a very high level it tries to do the following tasks:


  1. Allocate file structure and file descriptor<br>
  2. Associate file structure with file descriptor<br>
  3. Pathname lookup will return vnode<br>
  4. Associate vnode with file structure<br>
  5. Return file descriptor back to the user<br>

<br><br>

## Map - Open

All the system calls are mapped with system call number in <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/syscalls.master#L65" target="_blank">syscalls.master (kern)</a>.

| System Call Number | System Call | System Call Signature |
|:--------|:-------:|--------:|
| 5 | AUE_OPEN_RWTC | int open(char *path, int flags, int mode); |



More details about how system call is executed, can be found <a href="http://distkeys.com/operating%20systems/2014/08/03/system-call.html" target="_blank">here.</a>

<br><br>

## File Structure & File Descriptor

'Open' system call starts from function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1012" target="_blank">open(td, uap)</a> which calls function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1041" target="_blank">kern_open()</a> which finally calls function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1049" target="_blank">kern_openat().</a> This is the function from where all the action starts.


In function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1049" target="_blank">kern_openat()</a>, we are going to allocate the file descriptor. To do so we call function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1411" target="_blank">falloc()</a>.<br><br>

>Function falloc(struct thread *td, struct file **resultfp, int *resultfd) returns back pointer to the file entry and file descriptor.

<br>

### File Structure Allocation

Allocate space for file structure in kernel either by malloc or we use zalloc here because memory is assigned from file zone. There are multiple zones like file_zone, proc_zone or thread_zone. So, we ask memory from file zone and we pass the parameter which is space in this zone, if not available then are we willing to wait. Its a synchronous call. If memory is not available then thread will go to sleep if M_WAITOK is set.

{% highlight c linenos %}
     fp = uma_zalloc(file_zone, M_WAITOK | M_ZERO);
{% endhighlight %}


We get back pointer to file entry.
Zone limits are set during boot up time and whenever allocation or free happens it occurs from that limited allocation.
     
File structure looks as follows

{% highlight c linenos %}
// https://github.com/coolgoose85/FreeBSD/blob/master/sys/sys/file.h#L116

struct file {
     void              *f_data;     /* file descriptor specific data */
     struct fileops     *f_ops;          /* File operations */
     struct ucred       *f_cred;     /* associated credentials. */
     struct vnode      *f_vnode;     /* NULL or applicable vnode */
     short          f_type;          /* descriptor type */
     short          f_vnread_flags; /* (f) Sleep lock for f_offset */
     volatile u_int     f_flag;          /* see fcntl.h */
     volatile u_int      f_count;     /* reference count */
     /*
      *  DTYPE_VNODE specific fields.
      */
     int          f_seqcount;     /* Count of sequential accesses. */
     off_t          f_nextoff;     /* next expected read/write offset. */
     struct cdev_privdata *f_cdevpriv; /* (d) Private data for the cdev. */
     /*
      *  DFLAG_SEEKABLE specific fields
      */
     off_t          f_offset;
     /*
      * Mandatory Access control information.
      */
     void          *f_label;     /* Place-holder for MAC label. */
};
{% endhighlight %}     


We have created a file structure now we have to initialize it. So we, 

   - Increment the ref count for the file structure to 1.
   - Assign the input credentials.
   - Initialize the file ops to *badfileops* and we will update it later as we figure it out.
   - data is null
   - There are no *vnode* associated yet.
<br><br>

### File Descriptor 

Now, allocate the file descriptor. For that get lock and call <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1446" target="_blank">fdalloc()</a> to allocate fd.

{% highlight c linenos %}
// fdalloc()  https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1446 
if ((error = fdalloc(td, 0, &i))) {
{% endhighlight %}

Here ‘i' is the file descriptor to be returned.

Once we get file descriptor from function fdalloc() we associate the file descriptor to file structure we allocated above.

{% highlight c linenos %}
// fdalloc()  https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1453
p->p_fd->fd_ofiles[i] = fp;
{% endhighlight %}

Here p is the process structure where it maintains all the files opened.

Return the fd and file structure pointer from function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1411" target="_blank">falloc()</a> to function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1049" target="_blank">kern_openat()</a>

<br>

### How file descriptor is allocated (fdalloc)

 In function <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1331" target="_blank">fdalloc()</a>, we search the bitmap for a free descriptor. If we can’t find then grow the file table until limit is hit.

{% highlight c linenos %}
// fdalloc() https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/kern_descrip.c#L1352
for (;;) {
          fd = fd_first_free(fdp, minfd, fdp->fd_nfiles);
          if (fd >= maxfd)
               return (EMFILE);
          if (fd < fdp->fd_nfiles)
               break;
          fdgrowtable(fdp, min(fdp->fd_nfiles * 2, maxfd));
          }
          ...
{% endhighlight %}

<br><br>

## Call Tree

![center-aligned-image](/images/CallGraph-open.png)

<br><br>

## Pathname Lookup

Now we pick up the mode information for file provided as input. Mode value will be used if we have to create a file.

Now we call <a href="https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_vnops.c#L89" target="_blank">vn_open()</a> function and pass the input information.

{% highlight c linenos %}
// vn_open() https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_vnops.c#L89

error = vn_open(&nd, &flags, cmode, fp); 
{% endhighlight %}

Here, struct nameidata nd;<br>
If we don’t hit any error then we will get node related information in nd and all we do is extract the information and tie up with the fp file structure pointer.


<br><br>

## Associate vnode with file structure

If we don’t hit any error from vn_open() then we will get node related information in nd and all we do is extract the information and tie up with the *fp* file structure pointer.

{% highlight c linenos %}
Assign vnode https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1131
error = vn_open(&nd, &flags, cmode, fp); 
...
...
vp = nd.ni_vp;
fp->f_vnode = vp;
{% endhighlight %}


By this time if file operation is not set i.e. if it is still *badfileops* then we initialize it.

{% highlight c linenos %}
badfileops https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1138
if (fp->f_ops == &badfileops) {
          KASSERT(vp->v_type != VFIFO, ("Unexpected fifo."));
          fp->f_seqcount = 1;
          finit(fp, flags & FMASK, DTYPE_VNODE, vp, &vnops);
 }
{% endhighlight %}


So far, we have done all the book keeping operations. Once its done we need to write data to the file.
Now, we do the locking of a file. Based on what kind of locking is requested i.e.

  1. Exclusive lock<br>
  2. Shared Lock<br>
  3. Write Lock<br>
  4. Read Lock<br>
  5. Lease to client<br>

Lock the file, write to the file and set the attributes to the file.

{% highlight c linenos %}
file locking https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1145

if (flags & (O_EXLOCK | O_SHLOCK)) {
     …
     …
          if ((error = VOP_ADVLOCK(vp, (caddr_t)fp, F_SETLK, &lf,type)) != 0) {
               …
          }
     …
 }

if (flags & O_TRUNC) {
          if ((error = vn_start_write(vp, &mp, V_WAIT | PCATCH)) != 0)
               goto bad;
          VOP_LEASE(vp, td, td->td_ucred, LEASE_WRITE);
          …
          …
          vat.va_size = 0;
          …
          error = VOP_SETATTR(vp, &vat, td->td_ucred);
          …
}
{% endhighlight %}

<br><br>

## Return file descriptor back to the user

 Finally we return fd (indx) and 0 which is success from function kern_openat()
 
 {% highlight c linenos %}
 // Return fd https://github.com/coolgoose85/FreeBSD/blob/master/sys/kern/vfs_syscalls.c#L1184
 	td->td_retval[0] = indx;
	return (0);
 {% endhighlight %}
 
 
 <br> <br> <br> <br> <br>