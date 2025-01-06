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