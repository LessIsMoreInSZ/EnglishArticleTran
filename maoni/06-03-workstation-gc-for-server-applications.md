<h1>Workstation GC for server applications?</h1>
In Using GC Efficiently – Part 2 I talked about different flavors of GC that exist in the CLR and how you choose 
which flavor is good for your applications, and I said that the Server GC flavor is designed for server applications. 
As with any performance tuning there are always exceptions – there’s no one-rule-fits-all. Recently I worked with an internal group at MS on their 64-bit server scenario 
and it’s one of those situations where you do want to try Workstation GC for server applications so I thought it’d be interesting to talk about it here. 
The scenario is they are running 12 ASP.NET server processes on a 4-proc machine. Since they are using ASP.NET they get Server GC by default.  

 

First of all, let’s review how the server GC works. It creates a GC thread for each CPU you have on the machine and each GC thread collects parts of the GC heap. 
This means when you have 12 processes you get 48 GC threads. Imagine if all of the processes start doing GCs, there are 48 threads to schedule. In a managed process, 
the GC threads need to rendezvous at some points. 
Imagine if in the first process, one of the 4 threads happens to not be scheduled till the GC threads from the other 11 processes are scheduled, 
the first process will need to wait 11*4=44 quatums.

 

Keep in mind that it’s very common to run only one server process on a dedicated machine so you don’t get into this situation.
 But there are legitimate reasons when you do want to run multiple server processes on the same machine. For example, finacial cost for multiple machine, 
 adminitrative cost to manage multiple machines or all the servers need to share the same IP address (if they all server the same web site).

 

So when would you get into the situation where all processes start doing GCs at the same time? 
One of the common attributes of server applications is they try to use as much resource as possible on the machine. 
This is something that many people find hard to accept ‘cause we normally talk about how to make your app use less resource. 
When you dedicate a machine to running a specific application (+ the normal processes that the OS runs), 
you do want that app to use as much resource as possible otherwise the rest of the resource would be wasted. 
So normally server applications try to maximize their memory usage. 
As I mentioned in Using GC Efficiently – Part 1 one of reasons a GC would be triggered is when the machine is in low memory situation. 
So if you run multiple managed server applications they could all start to GC when the memory gets low. 
Note that this is not a specific problem to managed applications – native apps share the same problem 
because when the memory gets low they all start cleaning up processs which means they will all compete for CPU.

 

An absolute cricial thing to developing a server application is throttling. Unfortunately this is not known to everyone who develops server apps. 
Especially when developing managed server apps, some people think that GC should just take care of that aspect not realizing that GC manages memory, not requests. 
If your machine has 2GB of memory and each request takes 1MB of memory, it can’t handle more than 2048 requests concurrently. 
If you keep accepting requests as fast as they come in without any kind of throttling you will run out of memory when the number of requests exceeds what your machine can handle.

 

Some people think Server GC collects faster due to multiple GC threads working together so that would be one reason to choose Server GC over Workstation GC. 
This is true when you are collecting the same size of heap. Without throttling, however, 
you could have a bigger heap to collect because Server GC is more efficient (scalable on MP) so more time is spent accepting new requests 
if you accept much more concurrent requests you could end up using a lot more memory so you may not observe less time in GC simply because now GC has a lot more work to do.

 

Workstation GC with Concurrent GC off, on the other hand, runs on the user thread that triggered the GC and doesn’t create addition threads. 
When you run many processes on the same machine this can dramatically cut down the number of threads. As a matter of fact,
 our internal testing showed with many ASP.NET server processes running on one machine Workstation GC (no Concurrent GC) proved to be a good choice.

