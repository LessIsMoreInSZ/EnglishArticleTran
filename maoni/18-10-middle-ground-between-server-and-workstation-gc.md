<h1>Middle Ground between Server and Workstation GC</h1>

A long time ago I wrote about using Workstation GC on server applications when you have many instances of your server app running on the same machine. By default Server GC will treat the process as owning the machine so it uses all CPUs to do the GC work. More and more folks find themselves in a situation where they might have a few active instances of their server application running on the same machine. You could of course run each instance inside a container (a job object on Windows or cgroups on Linux) to limit resources that way. For example, if you limit the job object to only use 10 out of 40 CPUs on the machine, naturally Server GC will only create 10 GC threads.

To make it easier for folks to find a middle ground between Server and Workstation GC, in 4.6.2 I added a config to tell Server GC to use fewer CPUs than what’s on the machine. The rollup for Server 2012 R2 (KB4014604) describes this config (apparently this particular rollup was removed due to some issue with the installer and the new rollup does not describe the same info; I dunno why this is). In any case this is the text from that link:

————————————————— Fix 5

On a system that has many processors, there might a desire to have some processes use only some processes with server GC. For example if you have a 40-processor system, you might want to have process 0 use only 10 processors and process 1 use only 6 processors, so process 0 gets 10 GC heaps/threads and process 1 gets 6. Below is an example of using these Server GC configurations:

    <configuration>

        <runtime>

            <gcServer enabled=”true”/>

            <GCHeapCount enabled=”6″/>

            <GCNoAffinitize enabled=”true”/>

            <GCHeapAffinitizeMask enabled=”144″/>

        </runtime>

    </configuration>

<GCNoAffinitize enabled=”true”/> specifies to not affinitize the server GC threads with CPUs.

<GCHeapCount enabled=”6″/> 6 specifies that you want 6 heaps. The actual number of heaps is the minimum of the number of heaps you specify and the number of processors your process is allowed to use.

<GCHeapAffinitizeMask enabled=”144″/> This number is in decimal, so this is 0x90, which means using 2 bits as your process mask. The actual number of heaps is the minimum of the number of heaps you specify, the number of processors your process is allowed to use, and the number of set bits in the mask you specify.

—————————————————

By default Server GC threads are hard affinitized with their respective CPUs (we do this because it facilitates better cache usage). And since these configs became available I’ve seen quite a few folks use them but there seems to also be some confusion. So I will attempt to explain by illustrating with some sample scenarios.

Scenario 0

You have 4 processes that use Server GC on your 40-CPU machine and you want each process to use 10 CPUs and of course you want them to not overlap. So process #0 uses CPU 0 through 9; process #1 uses CPU 10 through 19 and so on. You’d want to specify your configs as the following for process #0:

    <gcServer enabled=”true”/>

    <GCHeapCount enabled=”10″/>

    <GCHeapAffinitizeMask enabled=”1023″/>

1023 is 0x3FF which means bits 0 through 9 are set. And specify 1047552 (0xFFC00) for process #1 and so on.

In general you do want to specify this mask; otherwise GC will just affinitize the Server GC threads with the first N CPUs (obviously you could have a scenario where you only have one process using Server GC and are fine with it using CPU 0 through N-1, in which case you don’t need to specify a mask value).

The value you specify for GCHeapCount means exactly that – the # of GC heaps. So this means there will be that many Server GC threads and that many Background Server GC threads (to do Background GCs) created.

Scenario 1

You simply don’t want to hard affinitize the Server GC threads for whatever reason.

    <gcServer enabled=”true”/>

    <GCNoAffinitize enabled=”true”/>

Scenario 2

You don’t want to hard affinitize the Server GC threads for whatever reason and want to limit the # of GC heaps to let’s say 10

    <gcServer enabled=”true”/>

    <GCHeapCount enabled=”10″/>

    <GCNoAffinitize enabled=”true”/>

If you specify GCNoAffinitize as true and a value for GCHeapAffinitizeMask, GCHeapAffinitizeMask will simply not be used.

If you do specify GCNoAffinitize as true, you should make sure your clr.dll is from the May 2018 rollup or newer (KB4103720 for Win2k16 and KB4098972 for Win2k12 R2) ‘cause there was a perf fix for it.

Note that these configs *only* affect the GC behavior – so the difference between specifying GCHeapCount as 10 on a 40-CPU machine and running the process in a job object that’s only allowed to use 10 CPUs is the former will still allow user threads to use all 40 CPUs whereas the latter will limit all threads in your process to only 10 CPUs.

