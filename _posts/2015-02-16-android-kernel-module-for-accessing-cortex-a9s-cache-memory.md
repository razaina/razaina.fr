---
layout: post
title: "Android kernel module for accessing Cortex A9's cache memory"
description: ""
permalink: "/:title"
category: 
tags: []
---
{% include JB/setup %}


# Objectives

The aim of this module was to be able to:

- enable the access to the Performance Monitor Unit (cf. <a
  href="http://infocenter.arm.com/help/topic/com.arm.doc.ddi0388i/BEHEDIHI.html"
  target="_blank">PMU</a>) control registers in user-mode
- read the Cycle Counter register
- trigger the cache flush operation from the external world (outside the kernel world)

# Get access to the Performance Monitor Unit

According to ARM's documentation: "The system control coprocessor, CP15, controls [...] cache
configuration and management." (cf. <a
href="http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0388f/CIHGCFGG.html"
target="_blank"> About system control</a>)

If we check the <a
href="http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0388f/CIHGCFGG.html"
target="_blank"> Registers summary</a> one can notice that we can access to the CP15 system control
registers using the instructions "MCR" and "MRC". 

"MRC" stands for "Move from ARM to Co-processor" and "MCR" for "Move from Co-Processor to ARM". 

    MRC{cond} <cp#>,<op>,<ARM srce>,<lhs>,<rhs>,{info}
    MCR{cond} <cp#>,<op>,<ARM dest>,<lhs>,<rhs>,{info}

where

    - <cp#> is the co-processor number (0-15)
    - <op> is the operation code required (0-7)
    - <ARM srce>/<ARM dest> is the ARM source/destination register (0-15)
    - <lhs> and <rhs> are co-processor register numbers (0-15)
    - {info} is optional extra information (0-7) 

(<a href="http://www.peter-cockerell.net/aalp/html/app-a.html" target="_blank">source</a>)

Still from the <a
href="http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0388f/CIHGCFGG.html"
target="_blank"> Registers summary</a> page, one can notice that one of the main functionalities of the CP15 is to provide "<a
href="http://infocenter.arm.com/help/topic/com.arm.doc.ddi0388i/CIHGBJID.html" target="_blank">Performance Monitor
Registers</a>".

From there one can see that while the CRn = C9, Op1 = 0, CRm = c13, Op2 = 0 we can access to the "Cycle Count
Register".

    uint32_t r = 0;

                      <cp#>   |<op> | <ARM Reg. dest.> | <lhs>|   <rhs> |  <op2>    (1)*
                    --------------------------------------------------------------------
    asm volatile("MRC p15,    |  0, |  %0,             |  c9, |   c13,  |  0",      "=r"(r)); 

    return r;

(*(1): <a href="https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html" target="_blank">6.43.2 Extended
Asm [...]</a>)

However, so far, the Cycle count is not incremented and furthermore it is not accessible from the
user-mode. Therefore, iwith the help of the manual regarding "<a
href="http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0388f/CIHGCFGG.html">4.2.24.
Performance Monitor Registers</a>" one should:

- Enable the Performance Monitor Control Regisers (PMCRs) in user-mode 
- Enable all counters by setting to "1" the first bit of the PMC Register (enable all counters)
- Enable the Cycle Count Register so it can be incremented and read


As I had the kernel source, after some grep I found out in the file "arch/arm/kernel/perf_event_v7.c" the functions that were doing what I was looking for. 

I just reused some part of the code that interested me and I've writtern the following functions:

{% highlight c %}
#define ARMV7_CCNT      31  /* Cycle counter */
#define ENABLE_CNTR (1 << ARMV7_CCNT)
#define ARMV7_PMNC_E        (1 << 0) /* Enable all counters */
#define ARMV7_CNTENS_C (1 << ARMV7_CCNT)

static void enable_cpu_counters(void* data)
{
    /* Enable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(1));
    /* Enable all counters */
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r"(ARMV7_PMNC_E));
    asm volatile("mcr p15, 0, %0, c9, c12, 1" : : "r" (ARMV7_CNTENS_C));
}

static void disable_cpu_counters(void* data)
{
    /* disable all counters */
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r" (0));
    asm volatile("mcr p15, 0, %0, c9, c12, 2" : : "r" (ARMV7_CNTENS_C));
    /* Disable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(0));
}

...

//then later when I want to enable the counters...
on_each_cpu(enable_cpu_counters, NULL, 1);

...

//or disable them
on_each_cpu(disable_cpu_counters, NULL, 1);
{% endhighlight %}

I embedded this code within a kernel module in order to enable the CPU cycle counter after inserting
the module using the command "insmod". The kernel module source code  is as follows:

{% highlight c %}
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>


#define ARMV7_CCNT      31  /* Cycle counter */
#define ENABLE_CNTR (1 << ARMV7_CCNT)
#define ARMV7_PMNC_E        (1 << 0) /* Enable all counters */
#define ARMV7_CNTENS_C (1 << ARMV7_CCNT)

static void enable_cpu_counters(void* data)
{
    printk("[+]" KERN_INFO "[" DRVR_NAME "] enabling user-mode PMU access on CPU #%d", smp_processor_id());
    /* Enable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(1));
    /* Enable all counters */
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r"(ARMV7_PMNC_E));
    asm volatile("mcr p15, 0, %0, c9, c12, 1" : : "r" (ARMV7_CNTENS_C));
}

static void disable_cpu_counters(void* data)
{
    printk(KERN_INFO "[" DRVR_NAME "] disabling user-mode PMU access on CPU #%d",smp_processor_id());
    /* Program PMU and disable all counters */
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r" (0));
    asm volatile("mcr p15, 0, %0, c9, c12, 2" : : "r" (ARMV7_CNTENS_C));
    /* Disable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(0));
}

int init_module(void)
{
    printk("[+] " KERN_INFO "[" DRVR_NAME "] Module initialisation\n");
    on_each_cpu(enable_cpu_counters, NULL, 1);
    return 0;
}

void cleanup_module(void)
{
    printk("[+]" KERN_INFO "[" DRVR_NAME "] Module unloading\n");
    on_each_cpu(disable_cpu_counters, NULL, 1);
}

MODULE_AUTHOR("razaina.");
MODULE_LICENSE("GPL");
MODULE_VERSION("1");
MODULE_DESCRIPTION("Enables user-mode access to ARMv7 PMU counters.);
{% endhighlight %}

Therefore on a higher level, from my Android native project (in user-mode) I can read the Cycle
Count register.
    
# Read the Cycle Count Register

Earlier in the previous section, I already showed the assembler instruction to execute in order to
get the Cycle Count Register.

Hereafter the function I use within my native project:

    static inline uint32_t
    get_cycle_count(void)
    {
            uint32_t r = 0;
            asm volatile("mrc p15, 0, %0, c9, c13, 0" : "=r"(r) );
            return r;
    }

# Trigger the flush operation

After grepping through the kernel sources, i found a file "arch/arm/include/asm/cacheflush.h" which ccontains the following functions:

    flush_icache_all() //Clean entire Instruction cache
    flush_kern_all() // Clean the entire cache (only L1 I guess..)
    flush_user_all() //Clean user space cache entries
    etc....


There was a particular function which was "flush_all_cpu_caches()" which refers to "flush_cache_all()". The latter actually refers to "__cpuc_flush_kern_all" which refers to the "flush_kern_all()" function. 

In the header of the current source file, there was the following description:

     *  The arch/arm/mm/cache-*.S and arch/arm/mm/proc-*.S files
     *  implement these methods.

In my case the corresponding files that contain the implementation (assembler) of these different functions was "arch/arm/mm/proc-arm926.S" and "arch/arm/mm/cache-v7".

Therefore by including the "asm/cacheflush.h" file in my kernel module, I could use the "flush_cache_all()" function.

However, my objective was to trigger the flush operation from the outside world, namely from my native project.

The following <a
href="https://smdaudhilbe.wordpress.com/2013/06/11/a-basic-character-driver-to-read-and-write-messages/"
target="_blank">tutorial</a> helped me for creating a "character driver" so I could do the following
steps in order to trigger any operation through my kernel module:
    
- my native program write a number (a character) within the device "/dev/mydriver"
- the kernel module is able to detect different events that is happening on "mydriver"
- the kernel module just read the written character and an operation is executed accordingly

The whole source code for the kernel module is as follows:
    
{% highlight c %}   
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/smp.h>
#include <asm/smp.h>
#include <asm/cacheflush.h>
#include <linux/proc_fs.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/slab.h> /* kmalloc() */
#include <linux/fcntl.h> /* O_ACCMODE */
#include <asm/system.h> /* cli(), *_flags */
#include <asm/uaccess.h> /* copy_from/to_user */
#include <linux/version.h>

#define PERF_DEF_OPTS (1 | 16)
#define PERF_OPT_RESET_CYCLES (2 | 4)
#define procfs_name "mydriver" // /dev/mydriver
#define FLUSHALL 3

#define ARMV7_CCNT      31  /* Cycle counter */
#define ENABLE_CNTR (1 << ARMV7_CCNT)
#define ARMV7_PMNC_E        (1 << 0) /* Enable all counters */
#define ARMV7_CNTENS_C (1 << ARMV7_CCNT)


#if !defined(__arm__)
#error Module can only be compiled on ARM machines.
#endif

extern long simple_strtol(const char *,char **,unsigned int);
void memory_exit(void);
int memory_init(void);
int memory_open(struct inode *inode, struct file *filp); 
int memory_release(struct inode *inode, struct file *filp); 
ssize_t memory_read(struct file *filp, char *buf, 
                    size_t count, loff_t *f_pos);
ssize_t memory_write( struct file *filp, char *buf,
                      size_t count, loff_t *f_pos);


/* Structure that declares the usual file */
/* access functions */
struct file_operations memory_fops = {
  read: memory_read,
  write: memory_write,
  open: memory_open,
  release: memory_release
};

/* Global variables of the driver */
/* Major number */
int memory_major = 69;
/* Buffer to store data */
char *memory_buffer;

int memory_open(struct inode *inode, struct file *filp) {
  /* Success */
    printk("[+]" KERN_INFO "[" DRVR_NAME "] Opening device !\n");
  return 0;
}

int memory_release(struct inode *inode, struct file *filp) {
  /* Success */
  return 0;
}

ssize_t memory_read(struct file *filp, char *buf, 
                    size_t count, loff_t *f_pos) { 
  /* Transfering data to user space */ 
  copy_to_user(buf,memory_buffer,1);

  /* Changing reading position as best suits */ 
  if (*f_pos == 0) { 
    *f_pos+=1; 
    return 1; 
  } else { 
    return 0; 
  }
}

ssize_t memory_write( struct file *filp, char *buf,
                      size_t count, loff_t *f_pos) {

  char *tmp;
  char *endptr;
  char ret;

  tmp=buf+count-1;
  copy_from_user(memory_buffer,tmp,1);
  ret = simple_strtol(memory_buffer, &endptr, 10);
  if(endptr == NULL)
  {
      printk("[+]" KERN_INFO "[" DRVR_NAME "] Failed to read an integer!\n");
  }
  else
  {
      if(ret == FLUSHALL)
      {
            printk("[+]" KERN_INFO "[" DRVR_NAME "] FLUSH CACHE ALL");
            flush_cache_all();
      }
      
  }
  return 1;
}

/* Called when a process tries to open the device file, like
 * "cat /dev/mycharfile"
 */
static int mydriver_device_open(struct inode *inode, struct file *file)
{
    printk("[+]" KERN_INFO "[" DRVR_NAME "] opening mydriver device.\n");
    return 0;
}

static void enable_cpu_counters(void* data)
{
    printk("[+]" KERN_INFO "[" DRVR_NAME "] enabling user-mode PMU access on CPU #%d", smp_processor_id());
    /* Enable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(1));
    /* Enable all counters */
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r"(ARMV7_PMNC_E));
    asm volatile("mcr p15, 0, %0, c9, c12, 1" : : "r" (ARMV7_CNTENS_C));
}

static void disable_cpu_counters(void* data)
{
printk("[+]" KERN_INFO "[" DRVR_NAME "] disabling user-mode PMU access on CPU #%d",
smp_processor_id());
    asm volatile("mcr p15, 0, %0, c9, c12, 0" : : "r" (0));
    asm volatile("mcr p15, 0, %0, c9, c12, 2" : : "r" (ARMV7_CNTENS_C));
    /* Disable user-mode access to counters. */
    asm volatile("mcr p15, 0, %0, c9, c14, 0" :: "r"(0));
}


int init_module(void)
{
    printk("[+]" KERN_INFO "[" DRVR_NAME "] Module initialisation\n");
    on_each_cpu(enable_cpu_counters, NULL, 1);

    int result = register_chrdev(memory_major, "mydriver", &memory_fops);
    if (result < 0) {
        printk(
          "[+]" KERN_INFO "[" DRVR_NAME "]: cannot obtain major number %d\n", memory_major);
        return result;
    }

      /* Allocating memory for the buffer */
      memory_buffer = kmalloc(1, GFP_KERNEL); 
      if (!memory_buffer) { 
        result = -ENOMEM;
        goto fail; 
      } 
      memset(memory_buffer, 0, 1);

      printk("[+]" KERN_INFO "[" DRVR_NAME "] Inserting memory module\n"); 
      return 0;

      fail: 
        memory_exit(); 
        return result;
        return 0;
}

void memory_exit(void) {
  /* Freeing the major number */
  unregister_chrdev(memory_major, "mydriver");

  /* Freeing buffer memory */
  if (memory_buffer) {
    kfree(memory_buffer);
  }

  printk("[+]" KERN_INFO "[" DRVR_NAME "] Removing module\n");

}


void cleanup_module(void)
{
    printk(KERN_INFO "[" DRVR_NAME "] Module unloading\n");
    on_each_cpu(disable_cpu_counters, NULL, 1);
    memory_exit();
}

MODULE_AUTHOR("razaina.");
MODULE_LICENSE("GPL");
MODULE_VERSION("1");
MODULE_DESCRIPTION("Enables user-mode access to ARMv7 PMU counters + Trigger the
flush_cache_all() function.");

{% endhighlight %}



<div id="disqus_thread"></div>
<script type="text/javascript">
/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
var disqus_shortname = 'razaina'; // required: replace example with your forum shortname

/* * * DON'T EDIT BELOW THIS LINE * * */
(function() {
    var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
    dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
    (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>



