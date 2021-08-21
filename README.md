# A Simple Character Device LKM

Your task is to implement a LKM device driver for a simple character device that supports reading, writing, and seeking. This assignment assumes you have completed the system call lab. You are advised to complete that lab before beginning this assignment.

## Creating Your Own Device Driver

Recall that if you want to add code to a Linux kernel, the usual method is to add some source files to the kernel source tree and recompile the kernel. This is what you did in the system call lab. After each change, the kernel must be recompiled, copied into the boot directory, and the computer must be rebooted. Again, you did this repeatedly in the system call lab. After you copied the kernel to the boot partition and rebooted, the changes that you made were installed in the kernel. If anything needed to be changed, this required you to repeat the whole process again. As you know from experience, this is tedious and inefficient.

### Building Loadable Kernel Modules (LKMs)

But from our readings, we know that we can also add code to the Linux kernel <i>while it is running</i>. A chunk of code that you add in this way is called a <i>loadable kernel module</i> (LKM). These modules can perform any function for the OS, but they have three typical uses:

- device drivers,
- filesystem drivers, and
- system calls.

The kernel isolates certain functions, including the modules (LKMs), especially well and therefore they don't have to be intricately wired into the rest of the kernel. The part of the kernel that is bound into the image that you boot (i.e., all of the kernel except the LKMs) is called the <i>base kernel</i>. LKMs communicate with the base kernel. There is a tendency to think of LKMs like user space programs. Modules do share a lot of user space program properties, but LKMs are definitely not user space programs. LKMs (when loaded) are very much part of the kernel. As such, they have free run of the system and can easily crash it.

LKMs have several advantages:

- You don't have to rebuild your kernel.
- They are space efficient because you can have them loaded only when you're actually using them.
- LKMs are much faster to maintain and debug.

In summary, LKMs are object files used to extend a running kernel's functionality. This is basically a piece of machine code that can be inserted and installed in the kernel on the fly without the need to reboot. This is very handy when you are trying to work with some new device and will be repeatedly be writing and testing your code. It is very convenient to write system code, install it, test it, and then uninstall it, without ever needing to reboot the system.

### Writing Source Code for a New Device Driver
Recall that the kernel uses <i>jump tables</i> in order to call the correct device drivers and functions of those drivers. Each LKM must define a standard jump table to support the kernels dynamic use of the module. The easiest way to understand the functionality that must be implemented is to create a simple module. We will create a new module `helloworld` that will log the functions being called. You should find the `hellomodule.c` file in the repository. Create a new directory called `helloworld` in the `/home/kernel/` directory. Open the `hellomodule.c` file in your editor.

This simple source file has all the code needed to install and uninstall an LKM in the kernel. There are two macros listed at the bottom of the source file that setup the jump table for this LKM. Whenever the module is installed, the kernel will call the routine specified in the `module_init` macro, and the routine specified in the `module_exit` will be called when the module is uninstalled. For writing your LKM, you should use the `helloworld` module as a template.

### Compiling Source Code for a New Device Driver

Let's now compile the module. There are a couple of ways to add our module to the list of modules to be built for a kernel. One is to modify the makefile used by the kernel build. The other is to write our own local makefile and attach it to the build when we want to make the modules. The latter is a bit safer, so what we'll do is create a new file called `Makefile` in the `/home/kernel/helloworld` directory, then type the following lines in the file:

```
obj-m:= hellomodule.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=/home/kernel/helloworld modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=/home/kernel/helloworld modules
```

Here, `obj-m` means that we are creating a module `hellomodule.o` from the source code file `hellomodule.c`, thus `hellomodule.c` should be in the same directory as your `Makefile`. In the `/home/kernel/helloworld` directory, run the following command:
```
make
```
You should now see the kernel object code `hellomodule.ko` in that directory.  In the following section, we will learn how to install our module into the kernel, then uninstall the module from the kernel. For writing your LKM, you should consider making a new directory in `/home/kernel/` for your Makefile and source code.

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

Each device will have a corresponding device file that is located in the `/dev` directory. If you list the files in that directory you will see all the devices currently known by the kernel. These are not regular files, they are <i>virtual files</i> that only supply data from or give data to the device. There is not any physical storage on the disk for these devices like the files that we are used to. The device driver code is responsible for creating a mechanism to store all the information.

To add a new device you need to create a new entry in the `/dev` directory. Each virtual file has information associated with it to tell the kernel which device driver to use when accessing the device. Recall that in Linux, each device driver is given a device \emph{major number} to uniquely identify it.

You can create a new virtual device file to be associated with your device driver. First, we'll find an unused major number for your kernel, then we'll use the `mknod` command to create a new device entry and associate the new major number for your device driver. 
```
sudo mknod  <location> <type of driver> <major number> <minor number>
```
The major number should be unique and you can look at current devices already installed, but usually user modules start at 240.  The <i>minor number</i> can be 0. The type of our driver should be `c` for character. Note that more sophisticated drivers must deal with blocks of arbitrary data, in which case the type is `b` (see `man mknod` for more details).  Putting it all together, run the following command:
```
sudo mknod /dev/simple_character_device c 240 0
```
Note that `simple_character_device` is a file, so we can change its permissions using `chmod`.  To be sure that we can read, write, and execute this file, run the following command:
```
sudo chmod 777 /dev/simple_character_device
```
Using `hellomodule.c` as template, in a new C file `my_driver.c`, your task is to create a new character device driver that suports the following file operations: 
```
open, read, write, llseek, release
```
which we will discuss in more detail momentarily.
You will need a character buffer is needed to store the data for this device. It will exist as long as the module is installed. Once it is uninstalled, all data will be lost. The buffer must be an array of 1024 characters.  In the next section, we discuss how we <i>register</i> our device driver.

## Registering/Unregistering the Device Driver
Recall that each device's virtual file is created with a major number associated with it, which allows the kernel to find the code that has been assigned to the major number when an applications tries to perform file operations upon it.
This means that when a module is loaded, it needs to tell the kernel which device it will be supporting by associating a major number with the module. Since our device is a character device, this can be accomplished using the `register_chrdev()` function, whose prototype is stated below:
```
int register_chrdev (unsigned int  major, 
			const char *  name, 
			const struct file_operations * fops);
```
Its first argument is self-explanatory, as is the second (recall that we called our virtual file `simple_character_device`, so we could pass that as our string).  The third parameter deserves a proper discussion which we will get to momentarily. We also need to unregister our device driver if we decide to unload the kernel module.  This is accomplished with the `unregister_chrdev(unsigned int major, const char* name)` function.


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
Except for its first member, all of its members are pointers to functions.  Each of these members supports some file operation functionality, and it is our responsibility to provide implementations of these operations (you can think of this struct like a Java interface in that it specifies the functionality that we the programmer must implement).
If we do not provide an implementation of any one of these functions, then the corresponding member is set to NULL by default, then the system will take care of the implementation of the function to give it some default functionality.  For this assignment, we won't provide implementations for the majority of these members.

You might have also noticed the `_t` suffix naming convention.  This stands for "type" and it is our makeshift C way of announcing what kind of data we should be expecting. There are no objects in C, so these are primitive data types, but it tells the programmer how to interpret the data.  For example, an implementation of `read` should return a value of "signed size type": if the output is positive, then it's a valid size; otherwise, it signals an error.

For the purposes of this assignment, our file operations struct, that we will pass as an argument to `register_chrdev()`, can be global and static. For example, if we were just concerned about implementing `read`, then this declaration would look something like this:
```
// recall that . is the member access operator in C
static struct file_operations simple_driver_fops =
{
    .owner   = THIS_MODULE,
    .read    = my_read,
};
```
where `my_read` is a function that implements `read`.  The prototype of our read function should of course be:
```
ssize_t my_read (struct file* , char* , size_t, loff_t* );
```
The declaration of the `THIS_MODULE` macro is contained in the `linux/module.h` header file. 

If we assign 0 to the major parameter, then the function will allocate a major device number (i.e. the value it returns) on its own, which allows us to not know in advance which major device numbers are taken.

## Implementing a Character Device Driver

Your main task is to give an implementation of five functions: `read`, `write`, `llseek`, `open`, `release`. You must name your functions:
```
my_read, my_write, my_llseek, my_open, my_release
```
The close functionality is handled by `release`. Since we are developing a very simple character device driver, many of the arguments to these functions that we need to implement will not be used.  Along these lines, do not overthink the `open` and `release` functions -- their implementations should be trivial. The nontrivial programming component of this assignment is the implementation of the following functions.

### `read` and `write`

- The read and write methods both perform a similar task, that is, copying data from and to application code. Therefore, their prototypes are pretty similar, and it’s worth introducing them at the same time:
```
ssize_t read(struct file *filp, char __user *buff, size_t count, loff_t *offp);
ssize_t write(struct file *filp, const char __user *buff, size_t count, loff_t *offp);
```
For both methods, `filp` is the file pointer and `count` is the size of the requested data transfer. The `buff` argument points to the user buffer holding the data to be written or the empty buffer where the newly read data should be placed. Finally, `offp` is a pointer to a "long offset type" object that indicates the file position the user is accessing. The return value is a "signed size type"; its use is discussed later. Let us repeat that the `buff` argument to the read and write methods is a <i>user space pointer</i>; therefore, it cannot be directly dereferenced by kernel code. Note that if the user attempts to write beyond the buffer, then you should write as many bytes to the buffer as possible, disregarding the remainder.

The return value for `read` is interpreted by the calling application program as follows:
- If the value equals the `count` argument passed to the read system call, the requested number of bytes has been transferred. This is the optimal case.
- If the value is positive, but smaller than `count`, only part of the data has been transferred. This may happen for a number of reasons, depending on the device. Most often, the application program retries the read. For instance, if you read using the `fread` function, the library function reissues the system call until completion of the requested data transfer. You can also see this behavior when you do a `strace` of `cat`.
- If the value is 0 , end-of-file was reached (and no data was read).
- A negative value means there was an error. The value specifies what the error was, according to `<linux/errno.h>`. Typical values returned on error include `-EINTR` (interrupted system call) or `-EFAULT` (bad address). For our LKM, the latter should be returned if any of the system calls invoked in `read` or `write` throws an error.

The return value for `write` is interpreted by the calling application program as follows:
- If the value equals `count`, the requested number of bytes has been transferred.
- If the value is positive, but smaller than `count`, only part of the data has been transferred. The program will most likely retry writing the rest of the data.
- If the value is 0, nothing was written. This result is not an error, and there is no reason to return an error code. Once again, the standard library retries the call to `write`. 
- A negative value means an error occurred; as for `read`, valid error values are those defined in `<linux/errno.h>`.

### `llseek`
The `llseek function` is called when one moves the cursor position within a file. The entry point of this method in user space is `lseek()`. One can refer to the `man` page in order to print the full description of either method from user space: `man llseek` and `man lseek`. Its prototype looks as follows:
```
loff_t(*llseek) (structfile *filp, loff_t offset, int whence);
```
- The return value is the new position in the file
- `loff_t` is an offset, relative to the current file position, which defines how much it will be changed
- `whence` defines where to seek from. In particular,
	- If `whence` is `SEEK_SET`, the cursor is set to `offset` bytes.
	- If `whence` is `SEEK_CUR`, the cursor is set to its current location plus `offset`.
	- If `whence` is `SEEK_END`, the cursor is set to the size of the file plus `offset`.
You must use these macros in your implementation. If adding the offset causes the cursor to become negative, then `-EINVAL` is returned, and the curstor remains unchanged.


### Hints

- If you are having trouble getting started, use `helloworld.c` as a starting point. 
- You will need to keep track of what header files are needed for implementing the five functions (these can be found by visiting their man pages). They will all be of the form `#include <linux/*.h>`.
- Remember that your module lives in <i>kernel space</i> but some arguments of our functions point to buffers in <i>user space</i>.
- Since we are implementing `llseek`, you will also need to keep track of the <i>present position</i> in the buffer. The `loff_t f_pos` field of the `file* filp` struct is essential for keeping track of this information. It represents the current reading or writing position. `loff_t` is a 64-bit value on all platforms
(`long long` in gcc terminology). The driver can read this value if it needs to know the current position in the file (i.e., your buffer). However, `read` and `write` should update the position using the pointer they receive as the last argument instead of acting on `filp->f_pos` directly. The purpose of the `llseek` method is to change the file position, which is why it will be modifying `filp->f_pos` directly. This is illustrated in the figure below.
![Screen Shot 2021-08-20 at 9 49 40 PM](https://user-images.githubusercontent.com/5934852/130309813-f673eebd-68ff-47fe-af13-359f1abb3900.png)
 - For more info on the `file` struct, visit https://docs.huihoo.com/doxygen/linux/kernel/3.7/structfile.html (even though most fields will not be used for our assignment).

## Testing

- For testing your device driver, you should modify the `seek` lab code so you can monitor the content of your device driver's buffer from user space. It is possible for read, write, and seek to appear functional in your C test code, but not interface properly with unix system utilities such as `echo`, `tail`, `head`, and so on. This means your implementation is incorrect, so be sure that your code interfaces with these utilities. Recall that `strace` allows you to see how these utilities are interfacing with your device. 

- An easy way to test the write functionality of your device driver is to perform I/O redirection in the terminal as follows:
```
echo 'hello world' > /dev/simple_character_device
```
which will write "hello world" to our character device driver. 
- To test the reading of characters from our device, try using the `cat` command, which will open, read repeatedly until EOF is reached, and then close the file.
```
cat /dev/simple_character_device
```
- To partially test `llseek` functionality you may use the `tail` system utility. In particular, you may look at `test.py` to see other test cases that will be used to test your code. To thoroughly test `llseek` you will need to write your own test program in C by modifying the `seek` lab. Remember, our character device is a <i>file</i>, so we can use familiar file I/O operations (e.g., `fopen`, `fclose`, `fseek`, etc..) for creating our test program. You are free to consult your notes or online tutorials about file I/O in C.

## Submission Instructions

You must follow these instructions carefully; otherwise, your solution will not interface with the script `test.py` and you will lose many points.

- `/home/kernel/<your last name>_PA1/`
	- `Makefile`: A makefile that will compile your LKM code (simply modify the Makefile in `/home/kernel/helloworld`).
	- `load.sh`: An executable `bash` script that will run the `mknod` and `insmod` commands to install your character device driver.
	- `unload.sh`: An executable `bash` script that will run the `rm` and `rmmod` commands to uninstall your character device driver.
	- `my_driver.c:` Your LKM driver code.

For submitting this assignment, create a zip file (use filename `<your last name>_PA1.zip`) and zip the directory above. When you double-click on your zip file, the contents should be a single directory. Submit that zip file as your submission in Moodle.

## Grading

Recall that 5 points of your assignment are determined by weekly updates. The remaining 95 points are determined by the grading script `test.py` which outputs how many of the 95 points you have earned. To ensure that you maximize your score on this assignment, you should write your assignment with respect to the following checkpoints.
	
### Checkpoint 1

Successfully install/uninstall device driver as LKM, then implement `open` and `release`. 
	
### Checkpoint 2

Implement `write`, then `read`.

### Checkpoint 3

Implement `llseek`. You cannot earn more than 80/95 if you do not attempt `llseek`.

## References

[1] Daniel Bovet and Marco Cesati. 2005. <i>Understanding The Linux Kernel</i>. Oreilly & Associates Inc.
[2] John Madieu. 2017. <i>Linux Device Drivers Development: Develop customized drivers for embedded Linux</i>. Packt Publishing.

