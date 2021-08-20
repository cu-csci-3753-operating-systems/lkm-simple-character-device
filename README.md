# A Simple Character Device LKM

## Creating Your Own Device Driver

If you want to add code to a Linux kernel, the usual method is to add some source files to the kernel source tree and recompile the kernel. This is what you did in the system call lab. After each change, the kernel must be recompiled, copied into the boot directory, and the computer must be rebooted. Again, you did this repeatedly in the system call lab. After you copied the kernel to the boot partition and rebooted, the changes that you made were installed in the kernel. If anything needed to be changed, this required you to repeat the whole process again. As you know from experience, this is tedious and inefficient.

### Building Loadable Kernel Modules (LKMs)

But from our readings, we know that we can also add code to the Linux kernel <i>while it is running</i>. A chunk of code that you add in this way is called a <i>loadable kernel module</i> (LKM). These modules can perform any function for the OS, but they have three typical uses:

- device drivers,
- filesystem drivers, and
- system calls.

The kernel isolates certain functions, including the modules (LKMs), especially well and therefore they don't have to be intricately wired into the rest of the kernel. The part of the kernel that is bound into the image that you boot (i.e., all of the kernel except the LKMs) is called the \emph{base kernel}. LKMs communicate with the base kernel. There is a tendency to think of LKMs like user space programs. Modules do share a lot of user space program properties, but LKMs are definitely not user space programs. LKMs (when loaded) are very much part of the kernel. As such, they have free run of the system and can easily crash it.

LKMs have several advantages:

- You don't have to rebuild your kernel.
- LKMs help you diagnose system problems. A bug in a device driver which is bound into the kernel can stop your system from booting at all.
- They are space efficient because you can have them loaded only when you're actually using them.
- LKMs are much faster to maintain and debug.


In summary, LKMs are object files used to extend a running kernel's functionality. This is basically a piece of machine code that can be inserted and installed in the kernel on the fly without the need to reboot. This is very handy when you are trying to work with some new device and will be repeatedly be writing and testing your code. It is very convenient to write system code, install it, test it, and then uninstall it, without ever needing to reboot the system.

### Writing Source Code for a New Device Driver
Recall that the kernel uses \emph{jump tables} in order to call the correct device drivers and functions of those drivers. Each LKM must define a standard jump table to support the kernels dynamic use of the module. The easiest way to understand the functionality that must be implemented is to create a simple module. We will create a new module `helloworld` that will log the functions being called. In Moodle you should find the `hellomodule.c` file. Create a new directory called `helloworld` in the `/home/kernel/` directory. Open the `hellomodule.c` file in your editor.

This simple source file has all the code needed to install and uninstall an LKM in the kernel. There are two macros listed at the bottom of the source file that setup the jump table for this LKM. Whenever the module is installed, the kernel will call the routine specified in the `module_init` macro, and the routine specified in the `module_exit` will be called when the module is uninstalled.

### Compiling Source Code for a New Device Driver

Let's now compile the module. There are a couple of ways to add our module to the list of modules to be built for a kernel. One is to modify the makefile used by the kernel build. The other is to write our own local makefile and attach it to the build when we want to make the modules. The latter is a bit safer, so what we'll do is create a new file called `Makefile` in the `/home/kernel/helloworld` directory, then type the following lines in the file:

```
obj-m:= hellomodule.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=/home/kernel/helloworld modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=/home/kernel/helloworld modules
```

Here, `obj-m` means that we are creating a module \texttt{hellomodule.o} from the source code file `hellomodule.c`, thus `hellomodule.c` should be in the same directory as your `Makefile`. In the `/home/kernel/helloworld` directory, run the following command:
```
make
```
You should now see the kernel object code `hellomodule.ko` in that directory.  In the following section, we will learn how to install our module into the kernel, then uninstall the module from the kernel.
### Installing/Uninstalling LKMs

To insert the module, type the following command:
```
sudo insmod hellomodule.ko
```
The kernel has tried to insert your module. If it is successful, the `module_init()` function will be called and you will see the log message that has been inserted into `/var/logs/system.log`. If you type `lsmod` you will see your module is now inserted in the kernel.

Let's now uninstall the module from the kernel by using the following command:
```
sudo rmmod hellomodule
```
To verify that the module was uninstalled, check the system log and you should see our module exit message. You can also use the `lsmod` command to verify the module is no longer in the system.




## Creating a Virtual File for the Device


We know that device drivers can be dynamically installed into the kernel, but how does the kernel know which device driver to use with which device? 

Each device will have a corresponding device file that is located in the \texttt{/dev} directory. If you list the files in that directory you will see all the devices currently known by the kernel. These are not regular files, they are \emph{virtual files} that only supply data from or give data to the device. There is not any physical storage on the disk for these devices like the files that we are used to. The device driver code is responsible for creating a mechanism to store all the information.

To add a new device you need to create a new entry in the \texttt{/dev} directory. Each virtual file has information associated with it to tell the kernel which device driver to use when accessing the device. Recall that in Linux, each device driver is given a device \emph{major number} to uniquely identify it.

You can create a new virtual device file to be associated with your device driver. First, we'll find an unused major number for your kernel, then we'll use the \texttt{mknod} command to create a new device entry and associate the new major number for your device driver. 
\begin{verbatim}
sudo mknod  <location> <type of driver> <major number> <minor number>
\end{verbatim}
The major number should be unique and you can look at current devices already installed, but usually user modules start at 240.  The \emph{minor number} can be 0. The type of our driver should be \texttt{c} for character. Note that more sophisticated drivers must deal with blocks of arbitrary data, in which case the type is \texttt{b} (see \texttt{man mknod} for more details).  Putting it all together, run the following command:
\begin{verbatim}
sudo mknod /dev/simple_character_device c 240 0
\end{verbatim}
Note that \texttt{simple\_character\_device} is a file, so we can change its permissions using \texttt{chmod}.  To be sure that we can read, write, and execute this file, run the following command:
\begin{verbatim}
sudo chmod 777 /dev/simple_character_device
\end{verbatim}
Using \texttt{hellomodule.c} as template, in a new \texttt{C} file \texttt{my\_driver.c}, your task is to create a new character device driver that suports the following file operations: 
\begin{center}
\texttt{open, read, write, llseek, release}
\end{center}
which we will discuss in more detail momentarily.
Immediately, we observe that a character buffer is needed to store the data for this device. It will exist as long as the module is installed. Once it is uninstalled, all data will be lost. In your previous courses, you have more than likely encountered many sophisticated implementations of buffers.  For this assignment we will not be concerned about such implementations. The buffer should be an array of 1024 characters.  In the next section, we discuss how we \emph{register} our device driver.


## Registering/Unregistering the Device Driver
Recall that each device's virtual file is created with a major number associated with it, which allows the kernel to find the code that has been assigned to the major number when an applications tries to perform file operations upon it.
This means that when a module is loaded, it needs to tell the kernel which device it will be supporting by associating a major number with the module. Since our device is a character device, this can be accomplished using the `register_chrdev()` function, whose prototype is stated below:
```
int register_chrdev (unsigned int  major, 
			const char *  name, 
			const struct file_operations * fops);
```
Its first argument is self-explanatory, as is the second (recall that we called our virtual file `simple_character_device`, so we could pass that as our string).  
The third parameter deserves a proper discussion which we will get to momentarily. We also need to unregister our device driver if we decide to unload the kernel module.  This is accomplished with the `unregister_chrdev(unsigned int major, const char* name)` function.


Below is the definition of the somewhat involved `file_operations` struct:
```
struct file_operations {
       struct module* owner;
       loff_t (*llseek) (struct file* , loff_t, int);
       ssize_t (*read) (struct file* , char* , size_t, loff_t* );
       ssize_t (*write) (struct file* , const char* , size_t, loff_t* );
       int (*readdir) (struct file* , void* , filldir_t);
       unsigned int (* poll) (struct file* , struct poll_table_struct* );
       int (*ioctl) (struct inode* , struct file* , unsigned int, unsigned long);
       int (*mmap) (struct file* , struct vm_area_struct* );
       int (*open) (struct inode* , struct file* );
       int (*flush) (struct file* );
       int (*release) (struct inode* , struct file* );
       int (*fsync) (struct file* , struct dentry * , int datasync);
       int (*fasync) (int, struct file* , int);
       int (*lock) (struct file* , int, struct file_lock* );
       ssize_t (*readv) (struct file* , const struct iovec* , unsigned long, loff_t* );
       ssize_t (*writev) (struct file* , const struct iovec* , unsigned long, loff_t* );
    };  
```
Except for its first member, all of its members are pointers to functions.  Each of these members supports some file operation functionality, and it is our responsibility to provide implementations of these operations (you can kind of think of this struct like a Java \texttt{interface} in that it specifies the functionality that we the programmer must implement).
If we do not provide an implementation of any one of these functions, then the corresponding member is set to \texttt{0} by default, then the system will take care of the implementation of the function to give it some default functionality.  For this assignment, we won't provide implementations for the majority of these members.

You might have also noticed the \texttt{\_t} suffix naming convention.  This stands for ``type" and it is our makeshift \texttt{C} way of announcing what kind of data we should be expecting. There are no objects in \texttt{C}, so these are primitive data types, but it tells the programmer how to interpret the data.  For example, an implementation of \texttt{read} should return a value of ``signed size type": if the output is positive, then it's a valid size; otherwise, it signals an error.

For the purposes of this assignment, our file operations struct, that we will pass as an argument to \texttt{register\_chrdev()}, can be global and static. For example, if we were just worried about implementing \texttt{read}, then this declaration would look something like this:
```
// recall that . is the member access operator in C
static struct file_operations simple_driver_fops =
{
    .owner   = THIS_MODULE,
    .read    = my_read,
};
```
where \texttt{my\_read} is a function that implements \texttt{read}.  The prototype of our read function should of course be:
```
ssize_t my_read (struct file* , char* , size_t, loff_t* );
```
\noindent The declaration of the \texttt{THIS\_MODULE} macro is contained in the \texttt{linux/module.h} header file. 


If we assign 0 to the major parameter, then the function will allocate a major device number (i.e. the value it returns) on its own, which allows us to not know in advance which major device numbers are taken.

\subsection{Implementing a Character Device Driver}

The close functionality is handled by \texttt{release}. Since we are developing a very simple character device driver, many of the arguments to these functions that we need to implement will not be used.  Along these lines, do not overthink the \texttt{open} and \texttt{release} functions -- their implementations should be trivial. The nontrivial programming component of this assignment is the implementation of 
\begin{enumerate}
\item \texttt{read} and \texttt{write}: The first parameter is a pointer to file structure. The second parameter is a pointer to a \emph{data buffer}. Note that these buffers will be different depending on whether we are reading or writing. The third parameter is the size of data to be read or written in bytes. The fourth parameter is a pointer to 64 bit current byte offset. The return value is the number of bytes actually read/written. Otherwise, \texttt{-1} is returned. You should return -1 for example if the user attempts to write beyond the buffer. On the other hand, you should return 0 if the user attempts to start reading beyond the buffer. Here, returning 0 indicates we have arrived at the ``end of the file". 
\item \texttt{llseek}: (man entry excerpt below)
```
#include <sys/types.h>
#include <unistd.h>

offset_t llseek(int fildes, offset_t offset, int whence);
```
The llseek function sets the 64-bit extended file pointer associated with the open file descriptor specified by fildes as follows:

\begin{itemize}
\item If whence is \texttt{SEEK\_SET}, the pointer is set to offset bytes.
\item If whence is \texttt{SEEK\_CUR}, the pointer is set to its current location plus offset.
\item If whence is \texttt{SEEK\_END}, the pointer is set to the size of the file plus offset.
\end{itemize}

Upon successful completion, llseek returns the resulting pointer location \emph{as measured in bytes from the beginning of the file}. Otherwise, -1 is returned, the file pointer remains unchanged. Here, ``pointer" is used generically, i.e., it is not a C pointer per se.

\item In addition to implementing these functions, your device driver must also print the number of times that the device has been opened to the kernel log.

\end{enumerate}
To see more info on these system calls, visit their man pages. Your task is to give an implementation of the five aforementioned functions. You must name your functions:
\begin{center}
\begin{verbatim}
my_read, my_write, my_llseek, my_open, my_release
\end{verbatim}
\end{center}

\subsubsection*{Hints}
\begin{itemize}
\item If you are having trouble getting started, use \texttt{helloworld.c} as a starting point. 
\item You will need to keep track of what header files are needed for implementing the five functions (these can be found by visiting their man pages).
\item Remember that your module lives in \emph{kernel space} but some arguments of our functions point to buffers in \emph{user space}. Check your notes to figure out how to get around this.
\item Since we are implementing \texttt{llseek}, you will also need to keep track of the \emph{present position} in the buffer. You might find the \texttt{loff\_t} \texttt{f\_pos} field of the \texttt{file* filp}  struct useful for keeping track of this information. It represents the current reading or writing position. \texttt{loff\_t} is a 64-bit value on all platforms
(\texttt{long long} in gcc terminology). The driver can read this value if it needs to know
the current position in the file (i.e., your buffer). \texttt{read} and \texttt{write}
should update the position using the pointer they receive as the last argument
instead of acting on \texttt{filp->f\_pos} directly. The one exception to this rule is in the
\texttt{llseek} method, the purpose of which is to change the file position.

 \href{https://docs.huihoo.com/doxygen/linux/kernel/3.7/structfile.html}{Click here} for more info about the \texttt{file} struct (even though the rest of its fields aren't that useful for our assignment).

\end{itemize}
\subsection{Testing}

We can do some ``quick and dirty" testing by doing I/O redirection in the terminal as follows:
\begin{verbatim}
echo 'hello world' > /dev/simple_character_device
\end{verbatim}
which will write ``hello world" to our character device driver. To test the reading of characters from our device, try using the \texttt{cat} command, which will open, read repeatedly until EOF is reached, and then close the file.
```
cat /dev/simple_character_device
```
However, to properly test \texttt{llseek()}, \emph{you will need to write your own test program} in \texttt{C} to test it. Remember, our character device is a \emph{file}, so we can use familiar file I/O operations (e.g., \texttt{fopen}, \texttt{fclose}, \texttt{fseek}, etc..) for creating our test program. Your test program should also test the functionality of \texttt{read} and \texttt{write}. You are free to consult your notes or online tutorials about file I/O in \texttt{C}.

\subsection{Submission Instructions}

For submitting this assignment, create a zip file (use filename: $<$your last name$>$\_PA1.zip) with all the files you have modified to create your new device driver. Submit that zip file as your submission in Moodle.


## Grading

Recall that 5 points of your assignment are determined by weekly updates. The remaining 95 points are determined by the grading script `test.py` which outputs how many of the 95 points you have earned. To ensure that you maximize your score on this assignment, you should write your assignment with respect to the following checkpoints.
	
### Checkpoint 1

Successfully install/uninstall device driver as LKM, then implement `open` and `release`. 
	
### Checkpoint 2

Implement `write`, then `read`.

### Checkpoint 3

Implement `llseek`
