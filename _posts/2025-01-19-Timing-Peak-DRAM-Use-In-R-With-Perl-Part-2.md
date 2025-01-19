---
title: "Profiling Peak DRAM Use in R With Perl - Part 2"
date: 2025-01-19
---

In the second part of this we implement the solution that was outlined at the end of [Part 1](https://chrisarg.github.io/Killing-It-with-PERL/2025/01/18/Timing-Peak-DRAM-Use-In-R-With-Perl-Part-1.html
):
1. utilize a Perl application that probes the operating system in real time for the RSS (Resident Set Size), i.e. the DRAM footprint of an application
2. fire the application from within R as a separate process, provide it with the PID (Process ID) of the R session and put it in the background
3. do the long, memory hungry application
4. upon the end of the calculation kill the Perl application and obtain DRAM usage for use within R

The Perl application that implements the first step is straightforward:
1. Obtain the command line arguments : the PID of the R session, a monitoring interval and a temporary file ...
2. for the Perl application to write out its own PID
3.  Obtain the initial memory usage by call the `ps` command line utility, using the excellent (and safe as it bypasses the shell!) [IPC::System::Simple](https://metacpan.org/pod/IPC::System::Simple) MetaCPAN module
5. Register an event handler that will write out the peak change ("\$max_delta") of the RSS from the initial value, as well as the initial value
6. Go into an infinite loop, in which
7.   * ps is probed for the _current_ value of the RSS,
     * subtracts it from the initial,
     * updates the value of \$max_delta if the current value over the baseline is larger
     * goes to sleep for a user defined period of time
     * reawakens at the top of the `while` loop

```perl
#!/usr/bin/perl

# Name: monitor_memory.pl
# Purpose: Registers peak DRAM usage of a process
# Usage: perl monitor_memory.pl <PID> <interval_in_seconds> <pidfile>
# Example: perl monitor_memory.pl 12345 1 /tmp/pidfile
# Date: January 19th 2025
# Author: Christos Argyropoulos
# License: MIT https://mit-license.org/

use strict;
use warnings;
use Time::HiRes qw(nanosleep); # high res waiting
use IPC::System::Simple qw(capturex); # safe capture that bypasses the shell

# Process the command line
my ( $pid, $interval_sec, $pidfile ) = @ARGV;
die "Usage: $0 <PID> <interval_in_seconds> <pidfile>\n"
  unless defined $pid && defined $interval_sec && defined $pidfile;

# Write our own PID that R will use to kill the Perl application
open( my $fh, '>', $pidfile ) or die "Can't write to $pidfile: $!";
print $fh "$$\n";
close $fh;

# Obtain initial memory usage
my $interval    = $interval_sec * 1_000_000_000;
my $initial_mem = capturex('ps', 'o', 'rss=', 'p', $pid);
chomp($initial_mem);
my $max_delta = 0;

# Register the INT and TERM signal handlers to print peak and initial DRAM usage
$SIG{INT} = $SIG{TERM} = sub {
    print "$max_delta\t$initial_mem\n";
    exit 0;
};


# Obtain the RSS, store the maximum delta up to this point, sleep and re-awaken
while (1) {
    my $current = capturex('ps', 'o', 'rss=', 'p', $pid);
    chomp($current);
    my $delta = $current - $initial_mem;
    $max_delta = $delta if $delta > $max_delta;
    nanosleep($interval);
}
```

Having delegated the peak DRAM monitoring to Perl, R is now free to obtain execution timings and memory allocation using Henrik Bengtsson's excellent `profmem` [package](https://cran.r-project.org/web/packages/profmem/index.html)
The package's vignette may be found [here](https://cran.r-project.org/web/packages/profmem/vignettes/profmem.html) and explains what id does and how this works (emphasis is mine):

>The `profmem()` function uses the `utils::Rprofmem()` function for logging memory 
allocation events to a temporary file. The logged events are parsed and returned
as an in-memory R object in a format that is convenient to work with. **All memory
allocations that are done via the native `allocVector3()` part of R's native API 
are logged**, which means that nearly all memory allocations are logged. 
Any objects allocated this way are automatically deallocated by R's garbage 
collector at some point. **Garbage collection events are _not_ logged by `profmem()`. 
Allocations _not_ logged are those done by non-R native libraries or R packages 
that use native code `Calloc()` / `Free()` for internal objects**. 
Such objects are _not_ handled by the R garbage collector.

Based on this description, `monitor_memory.pl` and `profmem` are complementary: the former will log peak running memory use, while the latter will log in all 
allocations, and the combination will provide the best of both worlds. 


The tricky part in the implementation is how to ensure that one does end up with an [orphan](https://en.wikipedia.org/wiki/Orphan_process) Perl process 
when the long calculation throws an error, and thus avoid having to kill the orphan somehow (Unix terminology is so violent). However, the Perl side should not
have to worry about the prospect of becoming an orphan, or detecting it has become one. Killing the Perl process (orphan or not), is a task for R, but Perl has
to help out, by providing R with the PID of the Perl application. Completion of the communication loop is a necessary, but not sufficient condition for culling
the orphan and R has to do its part when implementing the 2nd and 4th step of the top level logic. Here is where R's `tryCatch-finally` facilities come to shine:

```R
# Benchmarking allocations, peak DRAM use and execution timing of an expression in R
bench_time_mem<-function(x) {
	gc(reset=FALSE,verbose=FALSE)  ## force a gc here
	pidfile <- tempfile() # temp file to story Perl's PID
	outfile <- tempfile() # stdout redirection for Perl
	pid <- Sys.getpid()   # get R's PID
	ret<-NULL
	system2("./monitor_memory.pl", 
		c(pid,step_of_monitor,pidfile),wait=FALSE,stdout=outfile)
	Sys.sleep(0.2)  # Wait for PID file to be written
	monitor_pid <- readLines(pidfile)[1] # Get Perl's PID
	tryCatch (
		expr = {
			mem<-profmem(time<-system.time(x,gcFirst = FALSE))
			rettime<-c(time)
			names(rettime)<-names(time)
			retval<-c(time,"R_gc_alloc" = sum(mem$bytes,na.rm=T))
		}, # execute R expression, get timing and allocations
		finally = {
			system2("kill",c("-TERM", monitor_pid)) # kill the ? orphan
			Sys.sleep(0.2) # Wait for Perl to finish logging
			memstats<-read.csv(outfile,sep="\t",
				header=FALSE) # get memory statistics
			unlink(c(pidfile,outfile)) # cleanup files
			retval<-c(retval ,
				  "delta"= memstats[1,1]*1024,
				"initial"= memstats[1,2]*1024
				)
		}
	)

	return(retval)
}
```
The logic of the R partner is fairly straightforward:
1. Obtain the PID of the R process and a temporary file location for `monitor_memory.pl` to store its PID
2. Launch `monitor_memory.pl` and put it in the background
3. Proceed to execute the user provided expression in the `tryCatch` bloc and obtain allocations and execution timings with `mem<-profmem(time<-system.time(x,gcFirst = FALSE))`
4. Sum the total memory allocated during the execution of the user expression
5. Kill the Perl process in the `finally` block. This code will be executed even if errors are encountered in the user's expression, and hence no orphan will be allowed to live
6. Package the DRAM usage into the return value vector and send it to the user, along with allocations and timings. 

Let's look at a complete example written as a R script with command line parsing to see what we have achieved. For this example, we will use the two sequential and the
one large allocation we considered in [Part 1](https://chrisarg.github.io/Killing-It-with-PERL/2025/01/18/Timing-Peak-DRAM-Use-In-R-With-Perl-Part-1.html) :

```R
# Script Name: profile_time_memory.R
# Purpose: illustrate how to profile time and memory usage in R
# Usage: Rscript profile_time_memory.R --sleep_time 1 --size 1E6 --monitor_step 0.001
# Date: January 19th 2025
# Author: Christos Argyropoulos
# License: MIT https://mit-license.org/

library(profmem)
library(getopt)

# Define command line options
spec <- matrix(c(
  'sleep_time', 's', 1, "numeric", "Sleep time in seconds [default 1]",
  'size', 'n', 1, "numeric", "Size of the problem N [default 1E6]",
  'monitor_step', 'm', 1, "numeric", "Monitoring step in seconds [default 0.001]",
  'help', 'h', 0, "logical", "Show this help message and exit"
), byrow = TRUE, ncol = 5)

# Parse command line options
opt <- getopt(spec)


# Assign command line arguments to variables
if(is.null(opt$sleep_time)) opt$sleep_time<-1
if(is.null(opt$size)) opt$size<-1E6
if(is.null(opt$monitor_step)) opt$monitor_step<-0.001

sleep_time <- opt$sleep_time
N <- opt$size
step_of_monitor <- opt$monitor_step

busy_wait <- function(seconds) {
  start_time <- Sys.time()
  stop_time <- Sys.time()
  while (difftime(stop_time, start_time, units = "secs") < seconds) {
    stop_time <- Sys.time()
  }
}

bench_time_mem<-function(x) {
	gc(reset=FALSE,verbose=FALSE)  ## force a gc here
	pidfile <- tempfile() # temp file to story Perl's PID
	outfile <- tempfile() # stdout redirection for Perl
	pid <- Sys.getpid()   # get R's PID
	ret<-NULL
	system2("./monitor_memory.pl", 
		c(pid,step_of_monitor,pidfile),wait=FALSE,stdout=outfile)
	Sys.sleep(0.2)  # Wait for PID file to be written
	monitor_pid <- readLines(pidfile)[1] # Get Perl's PID
	tryCatch (
		expr = {
			mem<-profmem(time<-system.time(x,gcFirst = FALSE))
			rettime<-c(time)
			names(rettime)<-names(time)
			retval<-c(time,"R_gc_alloc" = sum(mem$bytes,na.rm=T))
		}, # execute R expression, get timing and allocations
		finally = {
			system2("kill",c("-TERM", monitor_pid)) # kill the ? orphan
			Sys.sleep(0.2) # Wait for Perl to finish logging
			memstats<-read.csv(outfile,sep="\t",
				header=FALSE) # get memory statistics
			unlink(c(pidfile,outfile)) #cleanup files
			retval<-c(retval ,
				  "delta"= memstats[1,1]*1024,
				"initial"= memstats[1,2]*1024
				)
		}
	)

	return(retval)
}



val<-bench_time_mem(
	{  	
		busy_wait(sleep_time);
		cat("\nAllocating\n")
		q<-rnorm(N);
		busy_wait(sleep_time);
		rm(q);gc(reset=F)
		q2<-rnorm(N);
		busy_wait(sleep_time);
		rm(q2);gc(reset=F)
	}
)

valcp<-bench_time_mem(
	{  
		busy_wait(sleep_time);
		cat("\nAllocating\n")
		q<-rnorm(N*2);
		busy_wait(sleep_time*2);
	}
)




cat("\n Allocating a double vector of length N = ",N," in R")
cat("\n            with busy waiting period  T = ",sleep_time," seconds")
cat("\n            and monitoring memory every : ",step_of_monitor," seconds")
cat("\n Will allocate N->gc->N and then 2N at once\n")
cat(paste(rep("=",65),collapse=""))
cat("\n Measured vs alloc'd in R  (2N at once): ", valcp["delta"]/valcp["R_gc_alloc"])
cat("\n Measured vs alloc'd in R  (N-> gc ->N): ", val["delta"]/val["R_gc_alloc"])
cat("\n")
cat("\n Performance of the two allocations : ")
cat("\n N-> gc ->N : \n")
print(val)
cat("\n 2N at once : \n")
print(valcp)
```

Running this from the command line we obtain the following output:
```text
Rscript --vanilla  profile_time_memory.R -s 1 -n 1000000 -m 0.00001

Allocating

Allocating

 Allocating a double vector of length N =  1e+06  in R
            with busy waiting period  T =  1  seconds
            and monitoring memory every :  1e-05  seconds
 Will allocate N->gc->N and then 2N at once
=================================================================
 Measured vs alloc'd in R  (2N at once):  0.995325
 Measured vs alloc'd in R  (N-> gc ->N):  0.4750328

 Performance of the two allocations : 
 N-> gc ->N : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       3.135        0.022        3.163        0.000        0.000 16210416.000 
       delta      initial 
 7700480.000 76742656.000 

 2N at once : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       3.088        0.017        3.110        0.000        0.000 16000048.000 
       delta      initial 
15925248.000 84443136.000 

```
As you can see the Perl monitor identified that the peak memory use of the sequential allocation-deallocation-allocation was half than of a single-step allocation of the entire workspace.
The Perl application's ability to monitor the process in very fine resolution (theoretically up to nanosec, but reallistically one microsecond is where I'd draw the limit), comes handy when one
has to monitor spikes in peak memory usage. Consider the example, in which some recently allocated memory is used for a fast task and then discarded (this is simulated by providing a small value to the `s` argument of the script). 
At low resolution, our estimates of the peak memory use are biased, e.g. :
```text
Rscript --vanilla  profile_time_memory.R -s .001 -n 1000000 -m 0.1

Allocating

Allocating

 Allocating a double vector of length N =  1e+06  in R
            with busy waiting period  T =  0.001  seconds
            and monitoring memory every :  0.1  seconds
 Will allocate N->gc->N and then 2N at once
=================================================================
 Measured vs alloc'd in R  (2N at once):  0.07372778
 Measured vs alloc'd in R  (N-> gc ->N):  0.3777522

 Performance of the two allocations : 
 N-> gc ->N : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       0.154        0.012        0.165        0.000        0.000 16210416.000 
       delta      initial 
 6123520.000 76742656.000 

 2N at once : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       0.108        0.011        0.119        0.000        0.000 16000048.000 
       delta      initial 
 1179648.000 84635648.000 

```

However changing the resolution of monitoring re-establishes accuracy:

```text
Rscript --vanilla  profile_time_memory.R -s .001 -n 1000000 -m 0.0001 
Allocating

Allocating

 Allocating a double vector of length N =  1e+06  in R
            with busy waiting period  T =  0.001  seconds
            and monitoring memory every :  1e-04  seconds
 Will allocate N->gc->N and then 2N at once
=================================================================
 Measured vs alloc'd in R  (2N at once):  0.995325
 Measured vs alloc'd in R  (N-> gc ->N):  0.4750328

 Performance of the two allocations : 
 N-> gc ->N : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       0.180        0.007        0.188        0.000        0.000 16210416.000 
       delta      initial 
 7700480.000 76746752.000 

 2N at once : 
   user.self     sys.self      elapsed   user.child    sys.child   R_gc_alloc 
       0.119        0.009        0.128        0.000        0.000 16000048.000 
       delta      initial 
15925248.000 84447232.000 

```

Having described the solution, let's provide some limitations and a context of use that acknowledges these limitations and some extensions:
* Small allocations (e.g. 100k doubles or below) will be invisible to the Perl monitor. This appears to be related to how the OS manages memory and how the kernel updates the page that is raided by `ps` for data
* This code is thus best used to monitor large allocations in long calculations
* One can extend the Perl monitor to take action with respect to R if memory usage grows at an unstainable rate, alert the user etc (important in my mind for large tasks executing remotely e.g. over the weekend). This is an interesting extension for future work
* One can easily extend the Perl to work under MacOs (trivial - as it has a `ps` command line utility), and Windows, e.g. run R under `WSL2` or use `tasklist` instead of `ps` (another possible extension)

I hope you enjoyed this journey with R and Perl so far! Have fun until the next time.    
