# locate/updatedb 
Your task is to code a multi-threaded version of the UNIX system utilities <tt>locate</tt> and <tt>updatedb</tt>. These programs were written well before the days of <i>solid state drives</i> (SSDs), which have become the de facto standard for secondary storage on modern machines. Not only do SSDs have very fast data access (generally less than 10 microseconds), but they also support <i>random access</i>. Note that a random-access memory device such as NVMe SSD allows data items to be read or written in almost the same amount of time irrespective of the physical location of data inside the memory, unlike traditional hard disk drives. On top of this, primary storage has also progressed by leaps and bounds since the 80's, so much so that it is feasible to have important databases reside in RAM. Your implementation will leverage these new technologies via multi-threading as to bring <tt>locate</tt> and <tt>updatedb</tt> into the 21st century. For more information on the virtues of multi-threaded disk access, see https://pkolaczk.github.io/disk-parallelism/.

Before you begin the assignment, you should first play around with <tt>locate</tt> and <tt>updatedb</tt> to understand their basic functionality and see how they are related to one another. At the very least, visit their <tt>man</tt> pages. You should also have working knowledge of <i>interprocess communication</i> (IPC) and multi-threading using POSIX <tt>pthreads</tt>.

## Specification

Your programs <tt>locate++</tt> and <tt>updatedb++</tt> must be written in the C programming language using POSIX <tt>pthreads</tt>. 

The program <tt>locate++ \<pattern\> \<num_threads\> </tt> must exhibit the following behavior. Typically <tt>\<pattern\></tt> would be <i>regular expression</i>, which would be . To keep things simple, we will support a set of common queries that we dub 
 * full queries: the entire filename is given, e.g., <tt>locate++ foo.txt 1 </tt>
 * prefix queries: a prefix of the filename is given, e.g., <tt>locate++ foo* 1 </tt>
 * suffix queries: a suffix of the filename is given, e.g., <tt>locate++ *.txt 1 </tt>.

0. Print a list of the userspace processes that are actively reading/writing/waiting on any locked file.
1. Check for file-related deadlock amongst the userspace processes that are currently running.
2. If there is no file-related deadlock amongst the userspace processes, print <tt>No deadlock.</tt> to standard out and exit. 
3. If deadlock has been detected, print <tt>Deadlock!</tt>, and then:
4. List each instance of deadlock on a new line and display the process names and file names involved.
5. Kill the process X involved in the most deadlocks and print <tt>Killed X.</tt> to the screen. 
6. Go to 1.

The program <tt>updatedb++ \<root_dir\> \<num_threads\></tt> must exhibit the following behavior.

  * If <tt>\<num_threads\></tt> is not specified, then the number of threads defaults to twice the number of CPU cores.
 1. Traverse the file system starting from <tt>\<root_dir\></tt>
  2. Create 

 Traditionally, the utility <tt>updatedb</tt> creates a database that resides on disk that <tt>locate</tt> queries, but we will be doing things a bit differently: the database will reside in RAM and the two processes will share this database so that <tt>locate++</tt> never has to read from the disk. In other words, <tt>updatedb++</tt> will serve as a daemon that runs in the background that <tt>locate++</tt> will query. To accomplish this, you must use some form of IPC. To keep things simple, <i>you may assume that only one copy of locate++ will run at a time</i>, which is most often the case we find ourselves in on say a desktop machine (allowing for many copies of locate++ would introduce yet another class of potential race conditions).
 
The most involved part of the assignment will be designing <tt>updatedb++</tt>. 
 
 0. Build a database starting from the specified \<root_dir\>.
 1. Wait for locate++ to signal that there is a query waiting to be processed.
 2. Once it recieves a signal, it procsses the query 
 3. Go to 1.
## Hints

This assignment requires you to obtain information about the , which will require a little research on your part. 
Once you have this information, the remainder of the assignment is algorithmic, so you may wish to revisit your notes on algorithms and data-structures. 

You should write the serial (i.e., single-threaded) version first! This should <i>always</i> be your first instinct when asked to code anything up in parallel -- it is simply too difficult to debug otherwise. In particular, see the checkpoints below for guidelines on how to incrementally write your program. Below are some miscellanious things to keep in mind while writing your program.

* Your IPC Lab solution should come in quite handy. 
* To keep the output legible, do not print absolute paths of files, just output their local file names.
* When going from Step 6 to Step 1, new processes might have kicked up, so we need to update the list of the userspace processes that are actively reading/writing/waiting on any locked file (just don't print this list again).
 * Think about what form of IPC should be used for locate++ and updatedb++ to communicate with each other.
 * You might need to invoke your program with <tt>sudo</tt> if you are indexing files and directories that can only be read by a super user.

## Grading

If your program does not compile or produce an executable called <tt>ddeadlock</tt> after running <tt>make</tt> on the provided <tt>Makefile</tt>, then you will receive a zero. You will be deducted 1 point per <tt>gcc</tt> warning, so do not throw flags to suppress warnings. Below is a detailed rundown of how you will be evaluated on this assignment.

### Documentation & Style (10 points)

If you are unsure about whether your practices in these two areas are acceptable, then defer to the Indian Hill style guidelines for the C programming language, or some other industrial standard that suits your liking.

If you are concerned about losing points here, then you should meet with your instructor during office hours to be sure things look right.

### I/O Formatting (10 points)

If you do not adhere to the input and output formatting conventions, then you will lose points. You are provided a black-box executable that is a perfect solution, so there is no ambiguity on the desired input and output.  If you leave debug print statements uncommented in your submission, then you will lose all 10 of these points. Your output should exactly match the output of the black-box, no exceptions.

If you are concerned about losing points here, you may run a <tt>diff</tt> on your output versus the black-box's output to be sure your output exactly matches (whitespace and all).

### Design (10 points)

This assignment is in the C programming language, so we are placing a premium on performance (imagine that this code will be executing on a server with many thousands of running userland processes). You will be docked points if you are too cavalier with system resources, or if parts of your implementation are too convoluted or clunky. If there is any unjustified hard-coding, e.g., placing false limits on the sizes of data-structures for processing input data, then you will lose these 10 points, as your program will crash for large enough input (even though the test cases may not show this). Finally, you will lose some points here if there are any egregious memory leaks, so don't forget your <tt>free</tt>'s.

If you are concerned about losing points here, then you should meet with your instructor during office hours to see if the part of your solution in question should be redesigned/simplified. 

### Correctness (70 points)

The correctness of your program will account for the majority of your grade. To ensure that you maximize this score, you should develop your code with respect to the milestones listed below.
 
#### Checkpoint 1 (30 points) 

Write the serial version of updatedb++.

#### Checkpoint 2 (35 points) 


#### Checkpoint 3 (50 points) 

Write locate++ (this should be simple -- don't overthink it). 
 
#### Checkpoint 4 (60 points) 

Steps 0-4, i.e., report whether a single deadlock has occurred or not occurred, print the process names and file names involved in that deadlock, kill any process involved in that deadlock, then repeat.

#### Checkpoint 5 (70 points)

Steps 0-6, i.e., report whether deadlock has occurred or not, and if so, list <i>many</i> ocurrences of deadlock and kill the process involved in the most ocurrences of deadlock, then repeat.

In principle, if you complete all 5 checkpoints, then you will earn full points; however, if any bugs in your code are made apparent by the test cases, then you will earn partial credit TBD. 


Finally, as always, you should focus on writing <i>correct</i> code first before you start making your code more efficient. 
