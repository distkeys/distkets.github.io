---
layout: post
title: "Virtual Memory - Part 4"
date: 2015-04-12 21:20
comments: true
categories:
 - Operating Systems

tags:
- '2015'
---

This is continuation of part series post of [Virtual Memory - Part 1](http://distkeys.com/operating%20systems/2015/01/02/virtual-memory-1.html), [Virtual Memory - Part 2](http://distkeys.com/operating%20systems/2015/01/02/virtual-memory-2.html), [Virtual Memory - Part 3](http://distkeys.com/operating%20systems/2015/01/02/virtual-memory-3.html)


{% include toc %}


This post is code walk of all the theory we went though in previous posts.


## Header files

All the virtual memory header files resides in *sys/vm* directory.

{% highlight c linenos %}
// 'vm_page' references page of virtual pages
//
// Each 'vm_page' structure exist for each page in the system and that is why
// 'vm_page' structure should be as small as possible
//
// Every 'vm_page' is referenced by only one object structure at a time
//
// vm_param.h - Machine independent virtual memory parameters
//
// - vm_map_object
// - vm_map_entry
// - vm_map
// - vmspace
// - vm_object
// - vm_page
//
/***********************
// vm.h is exported to the kernel as well as to user space
***********************/
typedef char vm_inherit_t;  /* inheritance codes */

// Shared mapping
#define VM_INHERIT_SHARE    ((vm_inherit_t) 0)
// Private copy
#define VM_INHERIT_COPY     ((vm_inherit_t) 1)
#define VM_INHERIT_NONE     ((vm_inherit_t) 2)
#define VM_INHERIT_DEFAULT  VM_INHERIT_COPY



typedef u_char vm_prot_t;   /* protection codes */

#define VM_PROT_NONE        ((vm_prot_t) 0x00)
#define VM_PROT_READ        ((vm_prot_t) 0x01)
#define VM_PROT_WRITE       ((vm_prot_t) 0x02)
#define VM_PROT_EXECUTE     ((vm_prot_t) 0x04)
#define VM_PROT_OVERRIDE_WRITE  ((vm_prot_t) 0x08)  /* copy-on-write */

#define VM_PROT_ALL     (VM_PROT_READ|VM_PROT_WRITE|VM_PROT_EXECUTE)
#define VM_PROT_RW      (VM_PROT_READ|VM_PROT_WRITE)
#define VM_PROT_DEFAULT     VM_PROT_ALL

// Object types
enum obj_type { OBJT_DEFAULT, OBJT_SWAP, OBJT_VNODE, OBJT_DEVICE, OBJT_PHYS,
                OBJT_DEAD };
// OBJT_DEFAULT - Zero fill areas on demand, heaps, stacks
// OBJT_SWAP    - When OBJT_DEFAULT needs to be paged then OBJT_DEFAULT is
//                promoted to OBJT_SWAP
// OBJT_VNODE   - For files
// OBJT_DEVICE  - Special case for OBJT_VNODE

/***********************
//vm_map.h
***********************/
/*
 *  Objects which live in maps may be either VM objects, or
 *  another map (called a "sharing map") which denotes read-write
 *  sharing with other maps.
 */
union vm_map_object {
    struct vm_object *vm_object;    /* object object */
    struct vm_map *sub_map;         /* belongs to another map */
};

// sub_maps here is used only for Kernel to supports things for user space.
// Kernel has memory and sub-map is part of that memory.
// Purpose of sub map to ensure not all the memory is used by one process but
// these days sub maps are not used that much as other subsystems have better
// way to ensure the resource utilization is optimum. Our focus is going to be
// on vm_object


// Map entry defines region in address space i.e. text, data etc.
struct vm_map_entry {
    // When page fault occurs we need to find out if address is valid part of
    //address space or not. If yes then the address for which page fault occur
    // where that page. prev and next are the pointers of link list.
    // Every time page fault occur the link list is traversed to determine if
    // address is valid and where it belongs.
    struct vm_map_entry *prev;  /* LL previous entry */
    struct vm_map_entry *next;  /* LL next entry */

    // Finding page during fault in linklist id inefficient. To improve we have
    // organized information also in Splay Tree. Insertion, lookup and removal
    // in Splay Tree is O(log n).
    struct vm_map_entry *left;  /* left child in binary search tree */
    struct vm_map_entry *right; /* right child in binary search tree */
    vm_offset_t start;          /* map entry start address of virtual address */
    vm_offset_t end;            /* map entry end address of virtual address */

    vm_offset_t avail_ssize;    /* how much max amt stack can grow if
                                   this is a stack */

    /* How much free space is between end of current entry and start of next
       entry in the node in splay tree */
    vm_size_t adj_free;         /* amount of adjacent free space */
    vm_size_t max_free;         /* max free space in Splay Tree */
    union vm_map_object object; /* object I point to */
    vm_ooffset_t offset;        /* offset into object */
    vm_eflags_t eflags;         /* map entry flags */
    vm_prot_t protection;       /* current protection code on the region */
    vm_prot_t max_protection;   /* what most allowed to do */
    vm_inherit_t inheritance;   /* inheritance */
    int wired_count;            /* can be paged if = 0 */
    vm_pindex_t lastr;          /* last read block*/
};



/* Behavior also useful for Garbage Collection */
#define MAP_ENTRY_BEHAV_NORMAL      0x0000  /* default behavior */
#define MAP_ENTRY_BEHAV_SEQUENTIAL  0x0040  /* expect sequential access */
#define MAP_ENTRY_BEHAV_RANDOM      0x0080  /* expect random access */
#define MAP_ENTRY_BEHAV_RESERVED    0x00C0  /* future use */




/*
 *  A map is a set of map entries.  These map entries are
 *  organized both as a binary search tree and as a doubly-linked
 *  list.  Both structures are ordered based upon the start and
 *  end addresses contained within each map entry.  Sleator and
 *  Tarjan's top-down splay algorithm is employed to control
 *  height imbalance in the binary search tree.
 *
 * List of locks
 *  (c) const until freed
 */
struct vm_map {
    struct vm_map_entry header; /* List of entries */
    struct sx lock;             /* Lock for map data */
    struct mtx system_mtx;      /* For short term locking here */
    int nentries;               /* Number of vm map entries hanging in list */
    vm_size_t size;             /* virtual size  of all entries */
    u_int timestamp;            /* Version number i.e. last modified time of
                                   this map */
    u_char needs_wakeup;
    u_char system_map;          /* (c) Am I a system map? */
    vm_flags_t flags;           /* flags for this vm_map */
    vm_map_entry_t root;        /* Root of a binary search tree */
    pmap_t pmap;                /* (c) Physical map */
    vm_map_entry_t deferred_freelist;
#define min_offset  header.start    /* (c) */
#define max_offset  header.end      /* (c) */
};





/*
 * Shareable process virtual address space.
 *
 * List of locks
 *  (c) const until freed
 */
struct vmspace {
    struct vm_map vm_map;   /* VM address map */
    struct shmmap_state *vm_shm;    /* SYS5 shared memory private data XXX */
    segsz_t vm_swrss;       /* resident set size before last swap */
    segsz_t vm_tsize;       /* text size (pages) XXX */
    segsz_t vm_dsize;       /* data size (pages) XXX */
    segsz_t vm_ssize;       /* stack size (pages) */
    caddr_t vm_taddr;       /* (c) user virtual address of text */
    caddr_t vm_daddr;       /* (c) user virtual address of data */
    caddr_t vm_maxsaddr;    /* user VA at max stack growth */
    int vm_refcnt;          /* number of references to vmspace */
    /*
     * Keep the PMAP last, so that CPU-specific variations of that
     * structure on a single architecture don't result in offset
     * variations of the machine-independent fields in the vmspace.
     */
    struct pmap vm_pmap;    /* private physical map */
};




/***********************
//vm_object.h
***********************/
struct vm_object {
    struct mtx mtx;
    TAILQ_ENTRY(vm_object) object_list; /* list of all objects */
    LIST_HEAD(, vm_object) shadow_head; /* objects that this is a shadow for */
    LIST_ENTRY(vm_object) shadow_list; /* chain of shadow objects */
    TAILQ_HEAD(, vm_page) memq; /* list of resident pages */
    vm_page_t root;             /* root of the resident page splay tree */
    vm_pindex_t size;           /* Object size */
    int generation;             /* generation ID */
    int ref_count;              /* How many refs?? */
    int shadow_count;           /* how many objects that this is a shadow for */
    objtype_t type;             /* type of pager */
    u_short flags;              /* see below */
    u_short pg_color;           /* (c) color of first page in obj */
    u_short paging_in_progress; /* Paging (in or out) so don't collapse or destroy */
    int resident_page_count;    /* number of resident pages */
    struct vm_object *backing_object; /* object that I'm a shadow of */
    vm_ooffset_t backing_object_offset;/* Offset in backing object */
    TAILQ_ENTRY(vm_object) pager_object_list; /* list of all objects of
                                                 this pager type */
    LIST_HEAD(, vm_reserv) rvq; /* list of reservations */
    vm_page_t cache;            /* root of the cache page splay tree */
    void *handle;
    union {
        /*
         * VNode pager
         *
         *  vnp_size - current size of file
         */
        struct {
            off_t vnp_size;
        } vnp;

        /*
         * Device pager
         *
         *  devp_pglist - list of allocated pages
         */
        struct {
            TAILQ_HEAD(, vm_page) devp_pglist;
        } devp;

        /*
         * Swap pager
         *
         *  swp_bcount - number of swap 'swblock' metablocks, each
         *           contains up to 16 swapblk assignments.
         *           see vm/swap_pager.h
         */
        struct {
            int swp_bcount;
        } swp;
    } un_pager;
};







/*
 *  Management of resident (logical) pages.
 *
 *  A small structure is kept for each resident
 *  page, indexed by page number.  Each structure
 *  is an element of several lists:
 *
 *      A hash table bucket used to quickly
 *      perform object/offset lookups
 *
 *      A list of all pages for a given object,
 *      so they can be quickly deactivated at
 *      time of deallocation.
 *
 *      An ordered list of pages due for pageout.
 *
 *  In addition, the structure contains the object
 *  and offset to which this page belongs (for pageout),
 *  and sundry status bits.
 *
 *  Fields in this structure are locked either by the lock on the
 *  object that the page belongs to (O) or by the lock on the page
 *  queues (P).
 *
 *  The 'valid' and 'dirty' fields are distinct.  A page may have dirty
 *  bits set without having associated valid bits set.  This is used by
 *  NFS to implement piecemeal writes.
 */

TAILQ_HEAD(pglist, vm_page);

struct vm_page {
    /* Pages are on all sorts of different lists like, active, inactive etc
     * pageq contains linkage to those */
    TAILQ_ENTRY(vm_page) pageq; /* queue info for FIFO queue or free list (P) */
    TAILQ_ENTRY(vm_page) listq; /* pages in same object (O)     */

    /* To look up page either browse the above list or use below pointers of
     * Splay Tree for quick look up */
    struct vm_page *left;       /* splay tree link (O)      */
    struct vm_page *right;      /* splay tree link (O)      */

    vm_object_t object;     /* which object am I in (O,P)*/

    vm_pindex_t pindex;     /* offset into object (O,P) - Page number of object */
    vm_paddr_t phys_addr;   /* physical address of page */
    struct md_page md;      /* machine dependent stuff */
    uint8_t queue;          /* page queue index */
    int8_t segind;          /* segment index */
    u_short flags;          /* see below */
    uint8_t order;          /* index of the buddy queue */
    uint8_t pool;
    u_short cow;            /* page 'copy on write' mapping count */

    /* wired down maps refs (P). If value is non zero then page is wired down.
    If two processes are accessing the same page then they increment the count
    so that page is wired down and when process don't need them anymore then
    they decrement the count */
    u_int wire_count;

    short hold_count;       /* page hold count i.e soft reference count */
    u_short oflags;         /* page flags (O), object flags */

    /* page usage/activity count. When page is being used actively we increment
       the count but when page is not being used that actively we start slowly
       bring down the count and when it gets to zero then we take the page away*/
    u_char  act_count;
    u_char  busy;           /* page busy count (O) */
    /* NOTE that these must support one bit per DEV_BSIZE in a page!!! */
    /* so, on normal X86 kernels, they must be at least 8 bits wide */
#if PAGE_SIZE == 4096
    u_char  valid;          /* map of valid DEV_BSIZE chunks (O) */
    u_char  dirty;          /* map of dirty DEV_BSIZE chunks */
#elif PAGE_SIZE == 8192
    u_short valid;          /* map of valid DEV_BSIZE chunks (O) */
    u_short dirty;          /* map of dirty DEV_BSIZE chunks */
#elif PAGE_SIZE == 16384
    u_int valid;            /* map of valid DEV_BSIZE chunks (O) */
    u_int dirty;            /* map of dirty DEV_BSIZE chunks */
#elif PAGE_SIZE == 32768
    u_long valid;           /* map of valid DEV_BSIZE chunks (O) */
    u_long dirty;           /* map of dirty DEV_BSIZE chunks */
#endif
};


/*
 * Each pageable resident page falls into one of five lists:
 *
 *  free
 *      Available for allocation now.
 *
 *  cache
 *      Almost available for allocation. Still associated with
 *      an object, but clean and immediately freeable.
 *
 *  hold
 *      Will become free after a pending I/O operation
 *      completes.
 *
 * The following lists are LRU sorted:
 *
 *  inactive
 *      Low activity, candidates for reclamation.
 *      This is the list of pages that should be
 *      paged out next.
 *
 *  active
 *      Pages that are "active" i.e. they have been
 *      recently referenced.
 *
 */



 // uma.h - Universal memory allocator
{% endhighlight %}


<br><br>
## Page Fault Reasons

{% highlight c linenos %}
#define PGEX_P          0x01     /* Protection violation vs. not present */
#define PGEX_W          0x02     /* during a Write cycle */
#define PGEX_U          0x04     /* access from User mode (UPL) */
#define PGEX_RSV     0x08     /* reserved PTE field is non-zero */
#define PGEX_I          0x10     /* during an instruction fetch */

{% endhighlight %}

<br><br>
## Data Members - vmspace

![center-aligned-image](/images/DataMembers-vmspace.png)

<br><br>
## mmap

> mmap -- allocate memory, or map files or devices into memory.
>
> mmap() creates a new mapping in the virtual address space of the calling process. The starting address for the new mapping is specified in addr. The length argument specifies the length of the mapping.


{% highlight c  %}
int mmap(struct thread *td, struct mmap_args *uap)

struct mmap_args {
    void *addr;
    size_t len;
    int prot;
    int flags;
    int fd;
    long pad;
    off_t pos;
};
{% endhighlight %}


<br><br>
### Example of mmap usage

{% highlight c  %}
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h> /* mmap() is defined in this header */
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

void err_quit(char *msg)
{
    printf("%s", msg);
    return;
}

// usage: a.out <fromfile> <tofile>
int main (int argc, char *argv[])
{
    int fdin, fdout;
    char *src, *dst;
    struct stat statbuf;
    int mode = 0x0777;

    if (argc != 3)
        err_quit ("usage: a.out <fromfile> <tofile>");

    /* open the input file */
    if ((fdin = open (argv[1], O_RDONLY)) < 0) {
        printf("can't open %s for reading", argv[1]);
        return 0;
    }

    /* open/create the output file */
    if ((fdout = open (argv[2], O_RDWR | O_CREAT | O_TRUNC, mode )) < 0) {
        printf ("can't create %s for writing", argv[2]);
        return 0;
    }
    /* find size of input file */
    if (fstat (fdin,&statbuf) < 0) {
        printf ("fstat error");
        return 0;
    }

    /* go to the location corresponding to the last byte */
    if (lseek (fdout, statbuf.st_size - 1, SEEK_SET) == -1) {
        printf ("lseek error");
        return 0;
    }

    /* write a dummy byte at the last location */
    if (write (fdout, "", 1) != 1) {
        printf ("write error");
        return 0;
    }

    /* mmap the input file */
    if ((src = mmap (0, statbuf.st_size, PROT_READ, MAP_SHARED, fdin, 0))
        == (caddr_t) -1) {
        printf ("mmap error for input");
        return 0;
    }

    /* mmap the output file */
    if ((dst = mmap (0, statbuf.st_size, PROT_READ | PROT_WRITE,
        MAP_SHARED, fdout, 0)) == (caddr_t) -1) {
        printf ("mmap error for output");
        return 0;
    }

    /* this copies the input file to the output file */
    memcpy (dst, src, statbuf.st_size);
    return 0;
} /* main */
{% endhighlight %}


<br><br>
## High Level Overview

- Check if we are mapping in uninitialized area, swap area or file read.
- Get file pointer based in file pointer provided as input to mmap using fget() which also serve as input verification.
- Get *vnode* associated with file to be mapped in memory.
- Call *vm_mmap()* function which does the real job in kernel as *mmap()* is just wrapper around *vm_mmap()*.
- As a part of *vm_mmap()*, we need to perform below jobs
    - Get *vm_page, vm_object, vm_map* where file areas can be mapped.
    - Get attributes for *vnode* provided
    - Allocate page for a given type
    - Allocate *vm_object* if required
    - Find *vm_map* in unallocated region in the target address map with the given length based on first-fit.
    - Once *vm_map* is found insert the entries into it.


<br>
![center-aligned-image](/images/mmap.png)
<br><br>


## MMAP - Code Walk

**mmap**

{% highlight c linenos %}
Function mmap(td, uap)
     struct thread *td;
     struct mmap_args *uap;
{

     addr = (vm_offset_t) uap->addr;
     size = uap->len;
     prot = uap->prot & VM_PROT_ALL;
     flags = uap->flags;
     pos = uap->pos;

     ...
     Sanity checking of inputs
     ...

     If we are mapping in uninitialized area
     ...

     If we are mapping in swap area
     ...

     If we are mapping file area
     get file pointer
     if ((error = fget(td, uap->fd, &fp)) != 0) {
          ...
     }

     ...
     ...

     Vode associated with file pointer
     vp = fp->f_vnode;

     Assign memory protection i.e. read, write, execute etc.
     if (vp->v_mount != NULL && vp->v_mount->mnt_flag & MNT_NOEXEC)
               maxprot = VM_PROT_NONE;
          else
               maxprot = VM_PROT_EXECUTE;
          if (fp->f_flag & FREAD) {
               maxprot |= VM_PROT_READ;
          } else if (prot & PROT_READ) {
               error = EACCES;
               goto done;
     }

     For shared memory, protection should be write
     if ((flags & MAP_SHARED) != 0) {
               if ((fp->f_flag & FWRITE) != 0) {
                    maxprot |= VM_PROT_WRITE;
                    ...
               }
          ...
     }

     handle = (void *)vp;
     handle_type = OBJT_VNODE;
     td->td_fpop = fp;

     error = vm_mmap(&vms->vm_map, &addr, size, prot, maxprot,
                        flags, handle_type, handle, pos);

     ...
     td->td_retval[0] = (register_t) (addr + pageoff);
     ...

}
{% endhighlight %}


mmap is just a wrapper around vm_mmap()

{% highlight c linenos %}
Function vm_mmap()
{
     ...
     Sanity Checking
     ...
     ...

     switch (handle_type) {
          case OBJT_DEVICE:
               ...
               ...
          case OBJT_VNODE:
               error = vm_mmap_vnode(td, size, prot, &maxprot, &flags,
                                          handle, foff, &object);
               break;
          case OBJT_SWAP:
               error = vm_mmap_shm(td, size, prot, &maxprot, &flags,
                                        handle, foff, &object);

               ....
     }

          Our case is OBJT_VNODE, vp = handle
          Function vm_mmap_vnode() {
               ...
               ...
               Get a ref on vnode
               vfslocked = VFS_LOCK_GIANT(mp);
               if ((error = vget(vp, LK_SHARED, td)) != 0) {
                    ..
               }

               obj = vp->v_object;
               type = OBJT_VNODE;
               handle = vp;
               ...
               ...

               Getattr for vnode
               error = VOP_GETATTR(vp, &va, cred)
               ...
               ...
               obj = vm_pager_allocate(type, handle, objsize, prot, foff);
                         Function vm_pager_allocate() {
                              This function allocates the instance of a pager of given type
                              ...
                              ...
                              ops = pagertab[type];
                              /*
                                   struct pagerops *pagertab[] = {
                                        ...
                                        &vnodepagerops,          /* OBJT_VNODE */
                                        ...
                                   }

                                   struct pagerops vnodepagerops = {
                                        .pgo_alloc =     vnode_pager_alloc,
                                        ...
                                        ...
                                   }
                              */

                              if (ops)
                                   ret = (*ops->pgo_alloc) (handle, size, prot, off);

                                   //So we are going to call vnode_pager_alloc()
                                        Function vnode_pager_alloc()
                                        {
                                             //Allocates pager for a vnode
                                             vp = (struct vnode *) handle;

                                             ...

                                             object = vm_object_allocate(OBJT_VNODE, OFF_TO_IDX(round_page(size)));
                                             ...
                                             vref(vp);
                                        } //end of vnode_pager_alloc()
                              }
                         } // end of vm_pager_allocate()

               vput(vp);
               VFS_UNLOCK_GIANT(vfslocked);

          } //end vm_mmap_vnode()


     So far we have allocated vm_page, vnode. Now, we need to allocate vm_map_entry and put it in
     appropriate data structure
     if (flags & MAP_STACK)
          rv = vm_map_stack(map, *addr, size, prot, maxprot,
              docow | MAP_STACK_GROWS_DOWN);
     else if (fitit)
          // Try to find it above tje addr
          rv = vm_map_find(map, object, foff, addr, size,
              object != NULL && object->type == OBJT_DEVICE ?
              VMFS_ALIGNED_SPACE : VMFS_ANY_SPACE, prot, maxprot, docow);
     else
          rv = vm_map_fixed(map, object, foff, *addr, size,
                     prot, maxprot, docow);

     ...
     ...
     ...
}
{% endhighlight %}


vm_map_find finds an unallocated region in the target address map with the given length.  The search is defined to be first-fit from the specified address; the region found is returned in the same parameter.


{% highlight c linenos %}
Function vm_map_find(vm_map_t map, vm_object_t object, vm_ooffset_t offset,
                          vm_offset_t *addr,     /* IN/OUT */
                          vm_size_t length, int find_space, vm_prot_t prot,
                          vm_prot_t max, int cow)
{
     Find space in a vm_map

     start = *addr;

     vm_map_findspace(map, start, length, addr)

               Find the first "fit" (lowest VM address) for "length" free bytes
               beginning at address >= start in the given map. This function returns
               the address (addr). This addr is not associated with vm_map entry.

               Function vm_map_findspace(vm_map_t map, vm_offset_t start, vm_size_t length,
                                               vm_offset_t *addr)
               {
                    We have min/max of VM addresses, request must be within that range

                    For the very first time when process boots up,
                    there are no blocks in VM address space for process
                    if (map->root == NULL) {
                         *addr = start;
                         goto found;
                    }

                    ...
                    ...
                    Find the entry into tree where we can fit our entry into vm_entry_map
                         while (entry != NULL) {
                              if (entry->left != NULL && entry->left->max_free >= length)
                                   entry = entry->left;
                              else if (entry->adj_free >= length) {
                                   *addr = entry->end;
                                   goto found;
                              } else
                                   entry = entry->right;
                         }
                         ...
                         ...
               } // end of vm_map_findspace()
     ...
     ...

     Insert into vm_map (vm_map.c)
     result = vm_map_insert(map, object, offset, start, start +
                                length, prot, max, cow);

     So, we have pager, address in the vm_map list where we can we can insert vm_map_entry
     but we have not yet created allocated vm_map_entry. So, function vm_map_insert()
     will create the vm_map_entry and insert into vm_map

          // Here start will have the addr value
          Function vm_map_insert(vm_map_t map, vm_object_t object, vm_ooffset_t offset,
           vm_offset_t start, vm_offset_t end, vm_prot_t prot, vm_prot_t max,
           int cow)
         {
              ...
              Sanity Checking
              ...

              Lookup entry in the tree. Since, we already have address in the tree all we are doing
              here is given the address find where we are in that respect. Are we ahead or behind?

              vm_map_lookup_entry(map, start, &temp_entry)

              ...
              ...

              If we have request of extending the object/vnode. Lets say if we malloc then we need to
              extend the initialized area. Function vm_object_coalesce() will try to grow the object
              if (.... || vm_object_coalesce(...)) {
                   ...
              }

              //Create a new vm_map_entry
              new_entry = vm_map_entry_create(map);

              Initialize the new_entry
              new_entry->start = start;
               new_entry->end = end;

               new_entry->eflags = protoeflags;
               new_entry->object.vm_object = object;
               new_entry->offset = offset;

               //Since its not a stack so size is 0
               new_entry->avail_ssize = 0;

               new_entry->inheritance = VM_INHERIT_DEFAULT;
               new_entry->protection = prot;
               new_entry->max_protection = max;
               new_entry->wired_count = 0;

               //Inser new vm_map_entry into list
               vm_map_entry_link(map, prev_entry, new_entry);

               //Enter the vm_map_entry in pmap which is machine dependent part
               vm_map_pmap_enter(map, start, prot,object, OFF_TO_IDX(offset), end - start,
                                     cow & MAP_PREFAULT_PARTIAL);

         } //end vm_map_insert()

} //end vm_map_find

{% endhighlight %}


One lock for active & inactive list<br>
- Active list<br>
- Inactive list<br>

One lock for cache and free list<br>
- Cache list - Pages moves from Inactive list to cache list<br>
- Free list - Pages moves from Cache list to free list<br>


<br><br>
## MMAP - Call Graph

![center-aligned-image](/images/Callsmmap.png)



<br><br><br><br>

