These namespaces were introduced in CLR 2.0. For example for the

 

GCHeap::GcCondemnedGeneration

 

symbol, it’s WKS::GCHeap::GcCondemnedGeneration for Workstation GC and SVR::GCHeap::GcCondemnedGeneration for Server GC
 (if you are reading the Investigating Memory Issues article in the recent MSDN magazine and are trying out some of the debugger commands mentioned in there).

 

If you are using CLR 1.1 or prior, the Workstation version lives in mscorwks.dll while the Server version lives in mscorsvr.dll 
so the symbol names are not prefixed with WKS:: or SVR::. So the breakpoint

 

bp mscorwks!WKS::GCHeap::RestartEE “j (dwo(mscorwks!WKS::GCHeap::GcCondemnedGeneration)==2) ‘kb’;’g'”

 

should be

 

bp mscorwks!GCHeap::RestartEE “j (dwo(mscorwks! GCHeap::GcCondemnedGeneration)==2) ‘kb’;’g'”

https://devblogs.microsoft.com/dotnet/not-seeing-the-wks-and-the-svr-namespace/

这些名称空间在CLR 2.0中引入。例如对于



GCHeap: GcCondemnedGeneration



符号，它是工作站GC的WKS::GCHeap:: gccondemnation generation和服务器GC的SVR::GCHeap:: gccondemnation generation
（如果您正在阅读最近的MSDN杂志上的调查内存问题文章，并且正在尝试其中提到的一些调试器命令）。



如果您使用的是CLR 1.1或更早版本，则工作站版本位于mscorsvr.dll中，而服务器版本位于mscorsvr.dll中
所以符号名没有前缀WKS:：或SVR:：。所以断点



bp mscorwks !WKS::GCHeap::RestartEE “ j (two (mscorwks!WKS::GCHeap:: gccondemnation generation)==2) ' kb '; ' g' ”



应该是



bp mscorwks !GCHeap::RestartEE " j (two （mscorwks!）GCHeap:: gccondemnation generation)==2) ' kb '; ‘ g’