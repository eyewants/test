#aos-team 11 (Project 3)

##Weighted Round-Robin (WRR)

WRR CPU Scheduling algorithm is based on the round-robin and priority scheduling algorithms. 
The WRR retains the advantage of round-robin in eliminating starvation and also integrates priority scheduling

#####Requirements of project 3
1.	Multiprocessor (SMP) must be supported 
2.	Default time slice is 10ms and weight range lies in between 1 and 20. 
3.	Periodic load balancing.
4.	Selecting a run-queue for a new task

##Implementation
####System calls  
1. sys_sched_setweight (376): sets the SCHED_WRR weight of process  
2. sys_sched_getweight (377): obtains the SCHED_WRR weight of a process  
3. sys_get_children_pid (378): obtain pid of the children process to set WRR as the default scheduling policy  
  
####Weighted round robin  
1.	Main algorithm is implemented in _kernel/sched_wrr.c_

2.	The priority of the scheduling class.  
		```  
    stop_sched_class → rt_sched_class → wrr_sched_class → fair_sched_class → idle_sched_class → NULL
		```  

3.	data structure  
  1)	struct sched_class wrr_sched_class  
  2)	struct wrr_rq : WRR- related fields in a runqueue   
      
4.	load-balance  
  
  1) Active balancing is performed regularly on each CPU, 500ms in this project  
 
  a. Registering IRQ  
    SCHED_WRR_SOFTIRQ is defined in interrupt.h and its handler, do_load_balance, is newly registered in kernel/sched.c as following.  
  
  ```
  open_softirq(SCHED_WRR_SOFTIRQ, do_load_balance);  
  ```
  
  b. Periodic call  
  
  timer_tick()  
  → update_process_times()   
  → scheduler_tick()  
  → trigger_load_balance_wrr()  
  → raise_softirq(SCHED_WRR_SOFTIRQ);  
  → do_load_balance  

  2)	Idle balancing  
  Not supported
  
  3) Selecting a run-queue for a new task  
  _select_task_rq_wrr()_ supports this feature

####Test Program(Trial)
1. Description  
	It calculates prime factorization number from 2 ~ 10000001 continuously  
2. Code location:
	```
	proj3-tizen/trial/trial.c
	```
3. How to compile the trial  
   1. Change directory to the code location, _proj3-tizen/trial_  
   2. Use the following option to compile  
		```	
		make trial
		```  
   3. In order to copy the trial to the device, run the sdb with a shell using following command.  
		```
		sbd push trial <target location>
		```
    Or,  use the following make option, copying the daemon into _/home/developer_ directory of the device.
		```
		make push
		```
	
####Dynamic Scheduling Policy Change(Systemd_wrr)
1. Description  
	It dynamically changes systemd and it's all child processes' scheduling class from SCHED_NORMAL to SCHED_WRR.
	For this operation, we have added additional system call, sys_get_children_pid.
	
2. Code location:  
	```
	proj3-tizen/trial/systemd_wrr.c
	```
3. How to compile the programs  
   1. Change directory to the code location, _proj3-tizen/trial_  
   2. Use following options to compile each program  
		```
		make systemd_wrr
		```
   3. In order to copy the programs to the device, run the sdb with a shell using following command.  
		```
		sbd push <application> <target location>
		```
    Or,  use the following make option, copying all three programs into _/home/developer_ directory of the device.  
		```
		make push
		```
	
####Scripts
1. cpu_on/off.sh: trun on/off cpus to configure the number of runnable cpu

	```
	proj3-tizen/linux-3.0/cpu_on.off
	```
	```
	  1 sdb root on
	  2 sdb shell 'echo 0 > /sys/devices/system/cpu/cpu1/online'
	  3 sdb shell 'echo 0 > /sys/devices/system/cpu/cpu2/online'
 	  4 sdb shell 'echo 0 > /sys/devices/system/cpu/cpu3/online'
 	  5 sdb shell 'cat /sys/devices/system/cpu/cpu0/online'
 	  6 sdb shell 'cat /sys/devices/system/cpu/cpu1/online'
 	  7 sdb shell 'cat /sys/devices/system/cpu/cpu2/online'
 	  8 sdb shell 'cat /sys/devices/system/cpu/cpu3/online'
 	  9 sdb shell 'echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor'
	  10 sdb shell 'cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor'
	```


##Discussion
####Investiageion
1. Track how long this program takes to execute with different weightings set and plot the result. You should choose a number to factor that will take sufficiently long to calculate the prime factorization of, such that it demonstrates the effect of weighting has on its execution time

	```
	To figure out the relation between the execution time of a process and its time slice, being propotional to the weight value, 
	We have run two differently weighted test programs(trial1 and trial2) and reach the the following conclusion: 
	1. The more weighted appliciton can complete its job first. 
	2. A very short time slice needs frequent scheduling(context switch), resulting in performance degradation
	3. If applications are assigned the same weight, i.e., the same time slice, they obviously results in the similar execution time. 
	4. When compared to the sigle core, dual (or more) core can make the job finished first; so, we can infer that Load balancing is crucial to the performace for multiprocessor systems.
	```
	
2. Test cases and results
1) single core with two processes
	```
	                     weight(1)      weight(20) 
	execution time(s)      34.55          17.22
	```
	```
	                     weight(10)     weight(20) 
	execution time(s)      27.549         21.47
	```
	```
	                     weight(20)     weight(20) 
	execution time(s)      27.538         27.543
	```	
	```
	                      weight(5)     weight(5) 
	execution time(s)      37.913         37.926
	```
2) dual core with two processes	
	```
	                      weight(1)     weight(20) 
	execution time(s)      20.41         14.072
	```	
3. Load balancing
  The following log messages shows that our implemenation for the load balancing works correctly! 

	```
  2320 [  197.389367] [AOS]CPU0's WEIGHT20:
  2321 [  197.389396] trial4(2499)'s weight(10)
  2322 [  197.389424] trial7(2501)'s weight(10)

  2324 [  197.389465] [AOS]CPU1's WEIGHT40:
  2325 [  197.389491] trial5(2500)'s weight(10)
  2326 [  197.389519] trial8(2506)'s weight(10)
  2327 [  197.389546] trial11(2509)'s weight(10)
  2328 [  197.389573] trial10(2512)'s weight(10)

  2330 [  197.389615] [AOS]CPU2's WEIGHT20:
  2331 [  197.389640] trial6(2498)'s weight(10)
  2332 [  197.389668] trial3(2494)'s weight(10)

  2334 [  197.389708] [AOS]CPU3's WEIGHT10:
  2335 [  197.389734] trial12(2511)'s weight(10)
	```

4. Time measurement
	We added time measurement code to the application
	```
	proj3-tizen/trial/trial.c
	```
	```
	gettimeofday(&after, 0);
	elapsed = (after.tv_sec-before.tv_sec)*1000000 + after.tv_usec-before.tv_usec;
	printf("%s(%d) exit:%f\n",argv[0], getpid(),elapsed/1000000.0);	
	```

