# locate/updatedb 
Your task is to code a multi-threaded version of the UNIX system utilities <tt>locate</tt> and <tt>updatedb</tt>. These programs were written well before the days of <i>solid state drives</i> (SSDs), which have become the de facto standard for secondary storage on modern machines. Not only do SSDs have very fast data access (generally less than 10 microseconds), but they also support <i>random access</i>. A random-access memory device such as NVMe SSD allows data items to be read or written in almost the same amount of time irrespective of the physical location of data inside the memory, unlike traditional hard disk drives. On top of this, primary storage has also progressed by leaps and bounds to the extent that it is feasible to have important databases persist in RAM. Your implementation will leverage these new technologies via multi-threading to give a 21st century implementation of <tt>locate</tt> and <tt>updatedb</tt>. For empirical data on multi-threaded SSD fetches, see https://pkolaczk.github.io/disk-parallelism/.

Before you begin the assignment, you should first play around with <tt>locate</tt> and <tt>updatedb</tt> and visit their <tt>man</tt> pages. You should also understand <i>interprocess communication</i> (IPC) and multi-threading using POSIX <tt>pthreads</tt>.

## Specification

Traditionally, the utility <tt>updatedb</tt> creates a database that resides on disk that <tt>locate</tt> queries, but we will be doing things a bit differently: the database will reside in RAM so <tt>locate++</tt> never has to directly access secondary storage. In other words, <tt>updatedb++</tt> will serve as a daemon that runs in the background that <tt>locate++</tt> will query. To keep things simple, you may assume that only one copy of `updatedb++` and `locate++` will run at a time, as allowing for multiple running instances of these programs would introduce even more concurrency issues. Your programs <tt>locate++</tt> and <tt>updatedb++</tt> must be written completely within in the C programming language using POSIX <tt>pthreads</tt>. 

### <tt>locate++</tt>
The program <tt>locate++</tt> must have the following prototype: <tt>locate++ -q \<pattern\> | -k</tt>.
Typically <tt>\<pattern\></tt> would be POSIX regular expression, but to keep things simple, you may assume that <tt>\<pattern\></tt> is in one of the following forms:
 * full query: the entire local filename is given, e.g., <tt>locate++ -q foo.txt</tt>,
 * prefix query: a prefix of the local filename is given, e.g., <tt>locate++ -q foo*</tt>,
 * suffix query: a suffix of the local filename is given, e.g., <tt>locate++ -q *.txt</tt>,
 * kill query: tell `updatedb++` to terminate itself, e.g., `locate++ -k`.
 
If the user does not adhere to the format above, print <tt>usage: locate++ -q \<pattern\> | -k</tt> to `stdout`. To simplify things, you may assume that <tt>updatedb++</tt> has been executed and that the database has been built before <tt>locate++</tt> executes. The program <tt>locate++</tt> should behave as follows:
 
 1. Send <tt>\<pattern\></tt> to <tt>updatedb++</tt>.
 2. <i>Immediately</i> print any match that <tt>updatedb++</tt> discovers.
 3. Exit.
 
### <tt>updatedb++</tt>
The program <tt>updatedb++</tt> must have the following prototype: <tt>updatedb++ \<root_dir\> \<num_threads\></tt>. The argument <tt>\<root_dir\></tt> is name of the root directory to be indexed. The argument <tt><num_threads\></tt> is a positive integer specified by the user. If <tt>\<num_threads\></tt> is not specified, then the number of threads defaults to 1. When <tt>updatedb++</tt> is executed, it should do the following:
 
 0. <b>Build</b> a database starting from the specified root directory using the specified number of threads.
 * Recall that the POSIX standard defines seven standard Unix file types: <i>regular, directory, symbolic link, FIFO special, block special, character special, and socket</i>. To keep things simple, we are only concerned with regular files and directories. You will need to read up a little on file I/O in C. 
 * Since we do not want `locate++` to access the disk, we need a data-structure to reside in memory that represents the database. Storing all of the absolute file names as an array of strings is not an option. File systems are inherently tree-like, so a natural choice is to use a tree-like data-structure to represent the database. You should build this data-structure top-down level-by-level in a mutli-threaded fashion using a <i>thread pool</i>. 
 1. <b>Wait</b> for locate++ to send a query.
 * This will require IPC. We do not want <tt>locate++</tt> and <tt>updatedb++</tt> to busy-wait under any circumstance, so your IPC solution must account for this.
 2. <b>Process</b> the query using the specified number of threads, then send the result back to locate++.
 * For processing the query, you must traverse the data-structure that represents the database using the specified number of threads with proper load-balancing.
 * Once a match is found, the thread should immediately send the relative filename to <tt>locate++</tt>.
 3. If <tt>locate++</tt> sent a kill query, then <b>Exit</b>; otherwise, Go to 1.
 
 ### Load Balancing
Load balancing always poses a challenge when writing multi-threaded programs. Step 0 is I/O bound, so the thread pool does a fine job of keeping the threads and CPU busy; however, Step 2 requires more cleverness to evenly distribute the work amongst the threads, as seen by the following example. Suppose we have 2 threads and that we are traversing a binary tree such that the left subtree has the vast majority of nodes. It may seem natural to assign the first thread to the left subtree and the second thread to the right subtree, but then the second thread will finish way before the first thread, which will be left holding the bag. This situation can be avoided by recollecting some elementary facts about traversal algorithms.

Recall that if we have any number of threads traversing a tree, each starting from the root, then at any point in the execution, a node is in precisely one of the following states: 
 * unmarked: The node has not been discovered by a thread.
 * marked: The node has been discovered by a thread.
 * processed: Every descendant of the node has been processed, where any <tt>NULL</tt> node is processed by default.

Intuitively, we want our threads to spread out and discover as many new nodes with as little rediscovery as possible, but note that sometimes a thread must rediscover a chain of unprocessed nodes in order to find more unmarked nodes. You should be able to tweak your data-structure just a little to allow your threads to traverse the tree in this fashion. If you choose some other data-structure to represent the database, then you will be responsible for coming up with your own strategy for load-balancing.
 
A POSIX mutex is what systems programmers call a "futex" (short for "fast userspace mutex"). The amount of time required to lock/unlock such a mutex is roughly 25ns which is about 4x faster than a typical fetch from RAM. This is quite efficient, to the point where you shouldn't shy away from using them if it could, for example, lead to better load-balancing solution.
 
## Hints

For some of us, this might be the first time we are doing a serious concurrent programming project. The cardinal rule of designing multi-threaded programs is to <i>always</i> write the serial (i.e., single-threaded) version first. Here, you may still use <tt>pthreads</tt> in your code, but with the understanding that there is precisely <i>one</i> thread so there will be no critical sections and thus no need for synchronization mechanisms. Once you get the serial version working, you should commit it to whatever version control software you are using so that you can always revert to something that works. See the checkpoints below for more detailed guidelines on how to incrementally write your multi-threaded program. 
 
 Below are some miscellanious things to keep in mind while writing your program.

 * `locate++` and `updatedb++` are not parent-child processes. Think about what IPC is most appropriate.
 * When `updatedb++` finds a match, the relative filename (with respect to the root directory of the database) of the matched file must be sent over to `locate++`. Think about introducing another pointer to tree data-structure to help you crawl up the tree to recover the relative filename when a match is found.    
 * Once a node is processed, by definition, no threads will ever traverse any of the nodes below it again. This should give you a clever way to "reset things" so that your tree data-structure will be ready for the next query.

## Grading

If your program does not compile or produce executables called <tt>locate++</tt> and <tt>updatedb++</tt> after running <tt>make</tt> on the provided <tt>Makefile</tt>, then you will receive at most a 30. You will be deducted 1 point per <tt>gcc</tt> warning, so do not ignore them. Below is a detailed rundown of how you will be evaluated on this assignment.

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

Write `locate++` (this should be simple -- don't overthink it). 
 
#### Checkpoint 4 (60 points) 

#### Checkpoint 5 (70 points)

#### Test Cases
You are encouraged to make your own small test cases. Download `random_dir.zip` for a test case of medium size, or download the directory structure of https://github.com/torvalds/linux for a large test case.

 
## Bonus

Unless you have done something clever, your algorithm for resolving queries probably amounts to a parallel brute-force search of a tree-like data-structure that represents the database. While this data-structure is required for reporting the <i>absolute</i> file names whose last name matches our pattern, it does not encode any meaningful information about the structure of <i>local</i> file names. Fortunately, there is a way to patch your tree-like data-structure with an additional data-structure(s) used for pattern matching that can quickly find which leaves (local file names) match the pattern. If you would like to trie your hand at some bonus, you will be awarded 10 extra points for a correct implementation of this faster data-structure.
 
 ## Conclusion
 
 We have made a number of simplifying assumptions along the way, and there is a lesson to be learned by removing each one of these assumptions. If you want to learn more, try removing these assumptions. 
 
 You may be wondering why developers have "left well-enough alone" and not bothered with a multi-threaded implementation of locate/updatedb, or any of these older system utilities, in standard Linux distributions. There are a number of good practical and theoretical reasons for this, which you should think about from a software engineering perspective. For example, when I try to <tt>locate</tt> a file that doesn't exist on my system, it takes about a second, which is a good response time (indeed, faster secondary storage also benefits <tt>locate/updatedb</tt>). Even though the average desktop user might not benefit tremendously from our multi-threaded implementation, there are certainly situations where our implementation would drastically improve performance, e.g., in scenarios where databases must be built quickly and are rapidly queried. The moral of the story is that in the real world, you should carefully evaluate your situation and determine if a multi-threaded solution is actually worth it. They are more prone to errors, and multi-threaded codebases are much more difficult to maintain. For these reasons, during the days of Moore's law, a common practice was to write correct serial programs and let the ever-improving CPU clock speeds make up for the performance down the line. Since the death of Moore's law, we can no longer depend on clock speed to bail us out, so CPU performance gains must be obtained by exploiting parallelism.  
 
A good solution to this assignment will demonstrate that you are proficient at writing concurrent userspace systems programs, which is a nice thing to have in your portfolio, but <b>do not post your solution publicly on GitHub.</b> If you would like to share this code with colleagues or potential employers, I suggest that you privately post to GitHub and provide them with a password.
