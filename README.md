# locate/updatedb 
Your task is to code a multi-threaded version of the UNIX system utilities <tt>locate</tt> and <tt>updatedb</tt>. These programs were written well before the days of <i>solid state drives</i> (SSDs), which have become the de facto standard for secondary storage on modern machines. Not only do SSDs have very fast data access (generally less than 10 microseconds), but they also support <i>random access</i>. A random-access memory device such as NVMe SSD allows data items to be read or written in almost the same amount of time irrespective of the physical location of data inside the memory, unlike traditional hard disk drives. On top of this, primary storage has also progressed by leaps and bounds to the extent that it is feasible to have important databases persist in RAM. Your implementation will leverage these new technologies via multi-threading to give a 21st century implementation of <tt>locate</tt> and <tt>updatedb</tt>. For more information on the virtues of multi-threaded disk access, see https://pkolaczk.github.io/disk-parallelism/.

Before you begin the assignment, you should first play around with <tt>locate</tt> and <tt>updatedb</tt> and visit their <tt>man</tt> pages. You should also understand <i>interprocess communication</i> (IPC) and multi-threading using POSIX <tt>pthreads</tt>.

## Specification

Traditionally, the utility <tt>updatedb</tt> creates a database that resides on disk that <tt>locate</tt> queries, but we will be doing things a bit differently: the database will reside in RAM so <tt>locate++</tt> never has to directly access secondary storage. In other words, <tt>updatedb++</tt> will serve as a daemon that runs in the background that <tt>locate++</tt> will query. To keep things simple, you may assume that only one copy of updatedb++ and locate++ will run at a time, as allowing for multiple running instances of these programs would introduce even more race conditions.


Your programs <tt>locate++</tt> and <tt>updatedb++</tt> must be written completely within in the C programming language using POSIX <tt>pthreads</tt>. 

In particular, the program <tt>locate++</tt> must have the following prototype: <tt>locate++ \<pattern\></tt>.
Typically <tt>\<pattern\></tt> would be POSIX regular expression, but to keep things simple, you may assume that <tt>\<pattern\></tt> is in one of the following forms:
 * full query: the entire local filename is given, e.g., <tt>locate++ foo.txt</tt>
 * prefix query: a prefix of the local filename is given, e.g., <tt>locate++ foo*</tt>
 * suffix query: a suffix of the local filename is given, e.g., <tt>locate++ *.txt</tt>.
 
It no <tt>\<pattern\></tt> is provided by the user, then print <tt>usage: locate++ \<pattern\></tt> to standard out.  If locate++ attempts to query the database before it has been built, then print <tt>database not ready!</tt> to standard out.


The program <tt>updatedb++</tt> must have the following prototype: <tt>updatedb++ \<root_dir\> \<num_threads\></tt>. The argument <tt>\<root_dir\></tt> is name of the root directory to be indexed. The argument <tt><num_threads\></tt> is a positive integer specified by the user. If <tt>\<num_threads\></tt> is not specified, then the number of threads defaults to 1. When <tt>updatedb++</tt> is executed, it should do the following:
 
 0. Build a database starting from the specified root directory using the specified number of threads.
 * Use a thread pool to traverse the directory structure level by level.
 * The POSIX standard defines seven standard Unix file types: <i>regular, directory, symbolic link, FIFO special, block special, character special, and socket</i>. To keep things simple, we are only concerned with regular files and directories. ou will need to read up a little on file I/O in C. <i>Write the serial version first!</i>
 * Recall that we do not want locate++ to access the disk, so we need a data-structure to reside in memory that represents the database. Simply storing all of the absolute file names as an array of strings is not an option for obvious reasons. The directory structure is inherently tree-like, so a natural choice is to use a tree-like data-structure to represent the database. You should build this data-structure top-down level-by-level in a mutli-threaded fashion using a <i>thread pool</i>. Think carefully about which parts of the tree 
 1. Wait for locate++ to signal that there is a query waiting to be processed.
 * Use a thread pool to traverse the directory structure level by level.
 * The POSIX standard defines seven standard Unix file types: <i>regular, directory, symbolic link, FIFO special, block special, character special, and socket</i>. To keep things simple, we are only concerned with regular files and directories. You will need to read up a little on file I/O in C. <i>Write the serial version first!</i>
 2. Once signaled, process the query using the specified number of threads, then report the result to locate++.
 * Use a thread pool to traverse the directory structure level by level.
 * The POSIX standard defines seven standard Unix file types: <i>regular, directory, symbolic link, FIFO special, block special, character special, and socket</i>. To keep things simple, we are only concerned with regular files and directories. You will need to read up a little on file I/O in C. <i>Write the serial version first!</i>
 3. Go to 1.
 
 
 
Load-balancing always poses a challenge when writing multi-threaded programs. Step 0 is I/O bound, so the thread pool does a fine job of keeping the threads busy; however, Step 2 requires some insight to avoid degenerating to a serial solution. If we have any number of threads traversing a tree, then recall from your algorithms notes that at any point a node is in precisely one of the following states: 
 * unmarked: The node has not been discovered by a thread.
 * marked: The node has been discovered by a thread.
 * processed: Every descendant of the node has been processed, where any <tt>NULL</tt> node is processed by default.

Intuitively, we want our threads to spread out and discover as many new nodes with as little rediscovery as possible, but note that sometimes a thread must rediscover a chain of unprocessed nodes in order to find more unmarked nodes. You should be able to tweak your data-structure just a little to allow your threads to traverse the tree in this fashion. If you choose some other data-structure to represent the database, then you will be responsible for coming up with your own strategy for load-balancing.
 
A POSIX mutex is what systems programmers call a "futex" (short for "fast userspace mutex"). The amount of time required to lock/unlock such a mutex is roughly 25ns which is about 4x faster than a typical fetch from RAM. This is quite efficient, to the point where you shouldn't shy away from using them if it could, for example, lead to better load-balancing solution.
 
## Hints


For some of us, this might be the first time we are doing a serious concurrent programming project. The cardinal rule of designing multi-threaded programs is to <i>always</i> write the serial (i.e., single-threaded) version first. Here, you may still use <tt>pthreads</tt> in your code, but with the understanding that there is precisely <i>one</i> thread so there will be no critical sections and thus no need for synchronization mechanisms. Once you get the serial version working, you should commit it to whatever version control software you are using so that you can always revert to something that works. See the checkpoints below for more detailed guidelines on how to incrementally write your multi-threaded program. 
 
 Below are some miscellanious things to keep in mind while writing your program.

 * The locate++ process will need to know the PID of updatedb++. This is a common problem when dealing with daemons that has many straightforward solutions, some more elegant than others, which is left for you to decide.
 * You should write a 
 * Once a node is processed, by definition, no threads will ever traverse any of the nodes below it again. This should help you "reset things" so your data-structure will be ready for the next query.
 * You should be able to easy convert your lab solutions to address some of the subproblems outlined above.

## Grading

If your program does not compile or produce an executable called <tt>ddeadlock</tt> after running <tt>make</tt> on the provided <tt>Makefile</tt>, then you will receive a zero. You will be deducted 1 point per <tt>gcc</tt> warning, so do not throw flags to suppress warnings. Below is a detailed rundown of how you will be evaluated on this assignment.

### Documentation & Style (10 points)
 
You will lose points if the definitions of your data-structures are mixed in with your other code. You should be refactoring your code along the way, do not save this task for the 11th hour.
 
Generally speaking, if you are unsure about whether your practices in these two areas are acceptable, then defer to the Indian Hill style guidelines for the C programming language, or some other industrial standard that suits your liking. 

If you are concerned about losing points here, then you should meet with your instructor during office hours to be sure things look right.

### I/O Formatting (10 points)

If you do not adhere to the input and output formatting conventions, then you will lose points. You are provided a black-box executable that is a perfect solution, so there is no ambiguity on the desired input and output.  If you leave debug print statements uncommented in your submission, then you will lose all 10 of these points. Your output should exactly match the output of the black-box, no exceptions.

If you are concerned about losing points here, you may run a <tt>diff</tt> on your output versus the black-box's output to be sure your output exactly matches (whitespace and all).

### Design (10 points)

This assignment is in the C programming language, so we are placing a premium on performance (imagine that this code will be executing on a server with many thousands of running userland processes). You will be docked points if you are too cavalier with system resources, or if parts of your implementation are too convoluted or clunky. If there is any unjustified hard-coding, e.g., placing false limits on the sizes of data-structures for processing input data, then you will lose these 10 points, as your program will crash for large enough input (even though the test cases may not show this). Finally, you will lose some points here if there are any egregious memory leaks.

If you are concerned about losing points here, then you should meet with your instructor during office hours to see if the part of your solution in question should be redesigned/simplified. 

### Correctness (70 points)

The correctness of your program will account for the majority of your grade. To ensure that you maximize this score, you should develop your code with respect to the milestones listed below.

#### Checkpoint 0 (30 points) 

Get two processes (that do not have the parent-child relation) talking to one another. Provide basic sanity-checking of input arguments to both programs.
 
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


As always, you should focus on writing <i>correct</i> code first before you start making your code more efficient. 

## Bonus and Conclusion

Unless you have done something clever, your algorithm for resolving queries probably amounts to a parallel brute-force search of a tree-like data-structure that represents the database. This data-structure is crucial since we must ultimately we must report the <i>absolute</i> file names whose last name matches our pattern, but it does not seem to encode any meaningful information about the structure of <i>local</i> file names. There is a way to modify your tree-like data-structure with an auxiliary data-structure(s) used for pattern matching that can quickly find which leaves (local file names) match the pattern. You will be awarded 10 bonus points for a correct implementation of this data-structure(s) that uses multi-threading.
 

 
 
