# A Simple Character Device LKM

## Creating Your Own Device Driver

If you want to add code to a Linux kernel, the usual method is to add some source files to the kernel source tree and recompile the kernel. This is what you did in the system call lab. After each change, the kernel must be recompiled, copied into the boot directory, and the computer must be rebooted. Again, you did this repeatedly when you added a system call. After you copied the kernel to the boot partition and rebooted, the changes that you made were installed in the kernel. If anything needed to be changed, this required you to repeat the whole process again. As you know from experience, this is tedious and inefficient.

### Building Loadable Kernel Modules (LKMs)

But from the readings, we know that we can also add code to the Linux kernel \emph{while it is running}. A chunk of code that you add in this way is called a \emph{loadable kernel module} (LKM). These modules can perform any function for the OS, but they have three typical uses:
\begin{itemize}
\item device drivers,
\item filesystem drivers, and
\item system calls.
\end{itemize}
The kernel isolates certain functions, including the modules (LKMs), especially well and therefore they don't have to be intricately wired into the rest of the kernel. The part of the kernel that is bound into the image that you boot (i.e., all of the kernel except the LKMs) is called the \emph{base kernel}. LKMs communicate with the base kernel. There is a tendency to think of LKMs like user space programs. Modules do share a lot of user space program properties, but LKMs are definitely not user space programs. LKMs (when loaded) are very much part of the kernel. As such, they have free run of the system and can easily crash it.

\noindent LKMs have several advantages:

- You don't have to rebuild your kernel.
- LKMs help you diagnose system problems. A bug in a device driver which is bound into the kernel can stop your system from booting at all.
- They are space efficient because you can have them loaded only when you're actually using them.
- LKMs are much faster to maintain and debug.


In summary, LKMs are object files used to extend a running kernel's functionality. This is basically a piece of machine code that can be inserted and installed in the kernel on the fly without the need to reboot. This is very handy when you are trying to work with some new device and will be repeatedly be writing and testing your code. It is very convenient to write system code, install it, test it, and then uninstall it, without ever needing to reboot the system.

### Writing Source Code for a New Device Driver
Recall that the kernel uses \emph{jump tables} in order to call the correct device drivers and functions of those drivers. Each LKM must define a standard jump table to support the kernels dynamic use of the module. The easiest way to understand the functionality that must be implemented is to create a simple module. We will create a new module \texttt{helloworld} that will log the functions being called. In Moodle you should find the \texttt{hellomodule.c} file. Create a new directory called \texttt{helloworld} in the \texttt{/home/kernel/} directory. Open the \texttt{hellomodule.c} file in your editor.

This simple source file has all the code needed to install and uninstall an LKM in the kernel. There are two macros listed at the bottom of the source file that setup the jump table for this LKM. Whenever the module is installed, the kernel will call the routine specified in the \texttt{module\_init} macro, and the routine specified in the \texttt{module\_exit} will be called when the module is uninstalled.

### Compiling Source Code for a New Device Driver

Let's now compile the module. There are a couple of ways to add our module to the list of modules to be built for a kernel. One is to modify the makefile used by the kernel build. The other is to write our own local makefile and attach it to the build when we want to make the modules. The latter is a bit safer, so what we'll do is create a new file called \texttt{Makefile} in the \texttt{/home/kernel/helloworld} directory, then type the following lines in the file:
\pagebreak

