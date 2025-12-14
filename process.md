1st - call the desired functuion in system call in user.h like this  (int trace(int); ) with  proper varibale pass
--
2nd -  entry the function in usys.pl  like this ( entry("trace"); ) 
--
End of user  side change <br>
3rd - add the user function in UPROGS like this ($U/_trace\) with proper line break in Makefile.
--
4th -  defiine the system call number like this ( #define SYS_trace  22  ) in syscall.h
--
5th - add the diclaration in syscall.c like this [SYS_trace]   sys_trace, in static uint64 (*syscalls[])(void)
--
6th - call  the systrace function with void in syscall.c like this extern uint64 sys_trace(void) 
--
7th - makke  the function in sysproc.c likke this  
	uint64   //7th change(adding)
	sys_trace(void)
	{
	  int sys_num;
	  //char abc[40];
	  argint(0, &sys_num);  //takes the 0 indexed argument inside 							sys_num variable
	  myproc() -> trace_num = sys_num;    // 8th change
	  return 0;
	}
--
8th - to declare a varibale global add the varible in proc.h 
--
9th - to change the system  with the desired output  add code in  the syscall function 
--
10th - to get what type of system call is called copy this static uint64 (*syscalls[])(void) = {
  function  and string/word like this [SYS_fork]    "fork",  
--
11th - properly declare  varibale like  this   p->trace_num = 0; 

