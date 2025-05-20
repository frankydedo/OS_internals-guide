# INDEX

- [LAB 1 - Options](#lab-1---options)
- [LAB 2 - System Calls and Virtual Memory Management](#lab-2---system-calls-and-virtual-memory-management)
  - [System Calls](#system-calls)
  - [Virtual Memory management](#virtual-memory-management)
- [LAB 3 - Lock and Condition Variable](#lab-3---lock-and-condition-variable)
  - [Theory](#theory)
  - [Implementation](#implementation)
    - [Lock](#lock)
    - [Condition variable](#condition-variable)

# LAB 1 - Options

Follow these steps to create a new config. For the sake of simplicity, we’ll use the **HELLO** configuration as an example:

- in _kern/main/main.c_ add the OPT:

        #if OPT_HELLO
        hello_function();
        #endif

- in _kern/main_ directory create the mfunc.c file, like so:

        #include <types.h>
        #include <kern/errno.h>
        #include <kern/reboot.h>
        #include <kern/unistd.h>
        #include <lib.h>
        #include <spl.h>
        #include <clock.h>
        #include <thread.h>
        #include <proc.h>
        #include <current.h>
        #include <synch.h>
        #include <vm.h>
        #include <mainbus.h>
        #include <vfs.h>
        #include <device.h>
        #include <syscall.h>
        #include <test.h>
        #include <version.h>
        #include "autoconf.h"  // for pseudoconfig


        #include "mfunc.h"

        void hello_function(void);

        void hello_function(void) {
            kprintf("Hello from OS161!\n");
        }

- in _kern/include_ create the mfunc.h file like so:

        // BY ME - lab 1

        #ifndef _MFUNC_H_       // if not defined
        #define _MFUNC_H_

        void hello_function(void);

        #endif

- in _kern/conf/conf.kern_ **APPEND** these lines:

        ########################################
        #                                      #
        #                lab_1                 #
        #                BY ME                 #
        #                                      #
        ########################################


        defoption hello
        optfile hello main/mfunc.c

  - **_defoption_**: defines a new option inside the kernel [MANDATORY]
  - **_optfile_**: here we can define which files must be **ignored** during the _build_ if the related option is **disabled** [CAN BE OMITTED if not required].

- in _kern/conf_ create a new file having the **same name as the configuration**, in this case will be **HELLO** (no extension).
  Edit it like so:

          # Kernel config file using dumbvm.
          # This should be used until you have your own VM system.

          include conf/conf.kern		# get definitions of available options

          debug				# Compile with debug info and -Og.
          #debugonly			# Compile with debug info only (no -Og).
          #options hangman 		# Deadlock detection. (off by default)

          #
          # Device drivers for hardware.
          #
          device lamebus0			# System/161 main bus
          device emu* at lamebus*		# Emulator passthrough filesystem
          device ltrace* at lamebus*	# trace161 trace control device
          device ltimer* at lamebus*	# Timer device
          device lrandom* at lamebus*	# Random device
          device lhd* at lamebus*		# Disk device
          device lser* at lamebus*	# Serial port
          #device lscreen* at lamebus*	# Text screen (not supported yet)
          #device lnet* at lamebus*	# Network interface (not supported yet)
          device beep0 at ltimer*		# Abstract beep handler device
          device con0 at lser*		# Abstract console on serial port
          #device con0 at lscreen*	# Abstract console on screen (not supported)
          device rtclock0 at ltimer*	# Abstract realtime clock
          device random0 at lrandom*	# Abstract randomness device

          #options net			# Network stack (not supported)
          options semfs			# Semaphores for userland

          options sfs			# Always use the file system
          #options netfs			# You might write this as a project.

          options dumbvm			# Chewing gum and baling wire.


          ###############lab_1################
          ###############By_ME################


          options hello           # BY ME: enables "opt hello"

- Once all these file editing operations are done, we should be able to recompile the kernel and its dependencies by running the following tasks from the _Command Palette_ _[control shift P]_, in this specific oreder:

        Run config              // type config's name (HELLO) when asked
        Make Depend
        Build and Install
        Run OS161

NOTE: if everything went well, the system must have generated the opt-hello.h file inside the _kern/compile/HELLO_ (automatically generated).

# LAB 2 - System Calls and Virtual Memory Management

## System Calls

### sys_read and sys_write

STEPS:

- in _kern/syscall_ create a new file and call it **file_syscall.c**. Here we'll implement sys_read() and sys_write().

  These two functions implement the write and read system calls for OS/161.

  - **sys_write** writes _nbytes_ characters from a user-space buffer _buf_ to the console (only supports stdout and stderr, i.e., file descriptors 1 and 2). It uses _copyin_ to safely transfer each byte from user space to kernel space, and putch to print to the console.
  - **sys_read** reads up to _nbytes_ characters from the console (only supports stdin, file descriptor 0) and stores them into the user-space buffer _buf_. It reads one character at a time using _getch_, then copies it back to user space with _copyout_. Reading stops if Enter (\n) is pressed.

  Both functions validate file descriptors and user pointers, returning appropriate error codes on failure.

  Implementation:

        #include <types.h>
        #include <kern/unistd.h> // per SYS_write e SYS_read
        #include <lib.h>
        #include <kern/errno.h>
        #include <syscall.h>
        #include <current.h>
        #include <copyinout.h>
        #include <uio.h>
        #include <proc.h>

        // NB:

        // ssize_t is defined in <include/types.h>
        // The ssize_t type in C is a signed data type commonly used in low-level
        // programming, particularly in system calls and file I/O operations. It is
        // defined in <include/types.h> and is typically used to represent the size of objects
        // or the result of operations that deal with sizes, such as reading or writing data.

        // userptr_t is defined in <include/types.h>
        // By using userptr_t, we can easily distinguish between:
        // - user-space pointers (userptr_t)
        // - kernel-space pointers (void \*)
        // In this case we should use userptr_t to represent pointers to user-space memory.

        ssize_t sys_write(int fd, userptr_t buf, size_t nbytes) {

        // Arguments:

        // fd: file descriptor (1 per stdout, 2 per stderr)
        // buf: buffer da scrivere
        // nbytes: numero di byte da scrivere

        // Solo stdout (1) e stderr (2) sono supportati
        if (fd != STDOUT_FILENO && fd != STDERR_FILENO) {
                return -EBADF;
        }

        // Verifica che il puntatore sia valido
        if (buf == NULL) {
                return -EFAULT;
        }

        size_t i;
        char ch;
        for (i = 0; i < nbytes; i++) {
                // Copia un byte dal buffer utente al kernel
                int result = copyin((const_userptr_t)((uintptr_t)buf + i), &ch, sizeof(char));

                if (result) {
                return -EFAULT;
                }

                putch(ch); // stampa su console
        }

        return (ssize_t)nbytes;

        }

        ssize_t sys_read(int fd, userptr_t buf, size_t nbytes){
        // Arguments:

        // fd: file descriptor (0 per stdin)
        // buf: buffer dove salvare i dati letti
        // nbytes: numero di byte da leggere

        // Solo stdin (0) è supportato
        if (fd != STDIN_FILENO) {
                return -EBADF;
        }

        // Verifica che il puntatore sia valido
        if (buf == NULL) {
                return -EFAULT;
        }

        size_t i;
        char ch;
        for (i = 0; i < nbytes; i++) {

                ch = getch(); // legge un carattere dalla console
                int result = copyout(&ch, (userptr_t)((uintptr_t)buf + i), sizeof(char));

                if (result) {
                return -EFAULT; // errore di copia
                }

                if (ch == '\n') {
                i++; // Aggiungi un byte in più per il carattere di nuova riga
                break; // Interrompe la lettura se viene premuto Invio
                }
        }

        return (ssize_t)i; // Restituisce il numero di byte effettivamente letti

        }

- now edit _kern/include/syscall.h_

  - add this include to enable _ssize_t_

        #include <types.h> /* BY ME: for ssize_t */

  - add functions prototype

        ssize_t sys_write(int fd, userptr_t buf, size_t nbytes);
        ssize_t sys_read(int fd, userptr_t buf, size_t nbytes);

- edit _kern/arch/mips/syscall/syscall.c_ by adding the corresponding CASEs inside the syscall() dispatcher function.

      // sys_write

      	// Arguments:

      	// 1st -> fd: file descriptor (1 per stdout, 2 per stderr)
      	// 2nd -> buf: buffer da scrivere
      	// 3rd -> nbytes: numero di byte da scrivere

      case SYS_write:
      err = sys_write(tf->tf_a0, (userptr_t)tf->tf_a1, tf->tf_a2);
      break;

      // sys_read

      	// Arguments:

      	// 1st -> fd: file descriptor (1 per stdout, 2 per stderr)
      	// 2nd -> buf: buffer da scrivere
      	// 3rd -> nbytes: numero di byte da scrivere

      case SYS_read:
      err = sys_read(tf->tf_a0, (userptr_t)tf->tf_a1, tf->tf_a2);
      break;

### sys\_\_exit()

NB: sys\_\_exit() has a **double undescore!**

**sys\_\_exit(int status)** terminates the current process. It destroys the user address space with _as_destroy()_ and stops the current thread using _thread_exit()_ , which never returns. If it does, _panic()_ is called. The _status_ is unused for now but would later be used for process exit codes.

The steps are preatty much the same as before.

- in _kern/syscall_ create a new file and call it **proc_syscall.c**

        #include <types.h>
        #include <kern/unistd.h>
        #include <clock.h>
        #include <copyinout.h>
        #include <syscall.h>
        #include <lib.h>
        #include <proc.h>
        #include <thread.h>
        #include <addrspace.h>

        /*
        * simple proc management system calls
        */
        void sys__exit(int status){    // Funzione di system call che termina il processo corrente

                /*
                * Recupera il puntatore alla address space (memoria utente) associata
                * al processo corrente. In OS161, ogni processo ha il suo "addrspace"
                * che rappresenta la memoria virtuale del processo.
                */
                struct addrspace *as = proc_getas();

                /*
                * Distrugge (libera) la address space.
                * Dealloca tutta la memoria utente allocata per il processo.
                * Dopo questa chiamata, il processo non ha più memoria utente valida.
                */
                as_destroy(as);

                /*
                * Termina il thread corrente.
                * Questa chiamata:
                * - Pulisce la struttura dati del thread
                * - Mette il thread nello stato "zombie"
                * - Fa in modo che il thread non sia più schedulabile
                *
                * ATTENZIONE: thread_exit() **NON ritorna** mai.
                * La funzione corrente viene interrotta.
                */
                thread_exit();

                /*
                * Se, per qualche motivo, thread_exit() dovesse ritornare
                * (cosa che **non dovrebbe MAI succedere**), andiamo comunque
                * in panic: significa che c'è stato un errore molto grave.
                */
                panic("thread_exit returned (should not happen)\n");

                /*
                * Questa istruzione serve solo per evitare warning del compilatore
                * tipo "variabile status non usata". Il valore `status` dovrebbe
                * in futuro essere salvato in una struttura dati, ma in questa
                * versione semplificata viene ignorato.
                */
                (void) status;
        }

- add the prototype in _kern/include/syscall.h_

        void sys__exit(int status);

- edit _kern/arch/mips/syscall/syscall.c_ by adding the corresponding CASE inside the syscall() dispatcher function.

        // sys_exit

                // Arguments:

                // 1st -> exitcode: codice di uscita del processo

        case SYS__exit:
        sys__exit((int)tf->tf_a0);
        break;

### Options

The final step here is to create a new configuration (e.g. **SYSCALLS**). This can be done by following the same steps as [LAB1](#lab-1).

## Virtual Memory management

Os161 base version comes with a 'dumb' implementation of virtual memory management which is only able to allocate physical memory in a contiguous way, without never releasing it. Our objective is to implement all the features nedded to achive the memory release.

NB: always remember to create a new configuration, in this case 'COREMAP'. (see [LAB 1](#lab-1---options))

- Firts step is to edit _arch/mips/vm/dumbvm.c_ file adding the declaration of all the needed variables.

  - \*coremap -> pointer to the bitmap
    - coremap[i] = 0 -> free
    - coremap[i] = 1 -> occupied
    - coremap[i] > 1 -> block partially allocated (offset from first)
  - total_frames -> number of total available frames in RAM (at bootstrap)
  - firstpaddr -> first available address in RAM
  - lastpaddr -> last valid address in RAM
  - coremap_initialized -> flag

  #if OPT_COREMAP
  static char \*coremap = NULL;
  static unsigned long total_frames = 0;
  static paddr_t firstpaddr, lastpaddr;
  static int coremap_initialized = 0;
  #endif

- Now in the same file edit the **vm_bootstrap** function to initialize coremap, like so:

        void
        vm_bootstrap(void)
        {
                #if OPT_COREMAP

                // LAB 2: bitmap coremap initialization

                // Primo indirizzo fisico disponibile
                firstpaddr = ram_stealmem(0);

                // Ultimo indirizzo fisico disponibile
                lastpaddr = ram_getsize();

                // Calcola numero di frame disponibili
                total_frames = (lastpaddr - firstpaddr) / PAGE_SIZE;

                // Calcola spazio necessario per la bitmap
                size_t coremap_bytes = total_frames * sizeof(char);

                // Dove posizionare la bitmap
                coremap = (char *)PADDR_TO_KVADDR(firstpaddr);
                bzero(coremap, coremap_bytes);

                // Sposta il primo indirizzo disponibile dopo la bitmap
                firstpaddr += ROUNDUP(coremap_bytes, PAGE_SIZE);

                coremap_initialized = 1;

                #endif

        }

- **getppages(npages)** implementation:

  - Purpose: Allocates npages contiguous physical memory pages (frames).
  - Used by: Both kernel (alloc_kpages) and user processes (as_prepare_load).
  - Returns: A physical address (paddr_t) of the first allocated frame, or 0 on failure.

        static
        paddr_t
        getppages(unsigned long npages)
        {
                #if OPT_COREMAP
                paddr_t addr;

                spinlock_acquire(&stealmem_lock);

                // If the coremap is not initialized yet, fall back to ram_stealmem
                if (!coremap_initialized) {
                        addr = ram_stealmem(npages);
                        spinlock_release(&stealmem_lock);
                        return addr;
                }

                // Search the coremap for npages contiguous free frames
                for (unsigned long i = 0; i <= total_frames - npages; i++) {
                        bool found = true;

                        // Check if all npages frames starting at i are free
                        for (unsigned long j = 0; j < npages; j++) {
                                if (coremap[i + j] != 0) {
                                        found = false;
                                        break;
                                }
                        }

                        // If a contiguous block is found
                        if (found) {
                                // Mark all npages frames as allocated (e.g., 1)
                                for (unsigned long j = 0; j < npages; j++) {
                                        coremap[i + j] = 1;
                                }

                                // Compute the physical address to return
                                addr = firstpaddr + i * PAGE_SIZE;

                                spinlock_release(&stealmem_lock);
                                return addr;
                        }
                }

                // If no contiguous block was found, return 0 (allocation failed)
                spinlock_release(&stealmem_lock);
                return 0;
                #endif

                // fallback sempre disponibile

                spinlock_acquire(&stealmem_lock);
                paddr_t addr = ram_stealmem(npages);
                spinlock_release(&stealmem_lock);
                return addr;
        }

- **alloc_kpages(npages)** implementation:

  - Purpose: Allocates npages kernel virtual memory pages.
  - How: Calls getppages, then converts the physical address to a virtual address using PADDR_TO_KVADDR.
  - Returns: A virtual address (vaddr_t) usable by the kernel.

        vaddr_t
        alloc_kpages(unsigned npages)
        {
                paddr_t pa;

                // Let the allocator sleep if memory is low (standard dumbvm behavior)
                dumbvm_can_sleep();

                // Request npages of contiguous physical memory
                pa = getppages(npages);

                // If allocation fails, return 0 (NULL address)
                if (pa == 0) {
                        return 0;
                }

                // Convert physical address to kernel virtual address and return it
                return PADDR_TO_KVADDR(pa);
        }

- **free_kpages(vaddr)** implementation:

  - Purpose: Frees memory previously allocated by alloc_kpages.
  - How: Converts the virtual address to physical and marks corresponding frames as free in the coremap.

        void
        free_kpages(vaddr_t addr)
        {
                #if OPT_COREMAP
                if (!coremap_initialized) {
                        // If the coremap is not ready, we can't free memory
                        return;
                }

                // Convert the virtual address to a physical address
                paddr_t pa = KVADDR_TO_PADDR(addr);

                // Calculate the frame index in the coremap
                unsigned long frame_index = (pa - firstpaddr) / PAGE_SIZE;

                spinlock_acquire(&stealmem_lock);

                // If we used a simple 1/0 bitmap, we can just scan forward until the allocation ends
                while (frame_index < total_frames && coremap[frame_index] == 1) {
                        coremap[frame_index] = 0; // Mark the frame as free
                        frame_index++;
                }

                spinlock_release(&stealmem_lock);
                #else
                        (void)addr;
                #endif
        }

# LAB 3 - Lock and Condition Variable

Before going through the steps required to address lab 3, let's first analyze all the theoretical aspect behind it.

## THEORY

First, let's understand why synchronization primitives are necessary.

Let's imagine a producer - consumer program where:

        THREAD 1

        while (!flag){;} // wait for x to be ready
        print(x);

and

        THREAD 2

        x = 100;        // produce x
        flag = true;    // set flag

This seems to work properly. The issues arise when the compiler, for several reasons, may decide to optimize the execution of _thread2_ by switching the two instructions, that from its point of view are totally independent and therefore can be reordered. This reordering can lead to unexpected behavior of our program, such as printing the wrong value of _x_.

This happens because we're trying to implement a synchronous behavior just by means of a simple variable (_flag_), which is not designed for enforcing ordering or **mutual exclusion**. To ensure correct behavior, we need dedicated constructs specifically built to implement synchronous programming: **synchronization primitives**.

There are three possible approaches:

- Memory barriers
- Hardware instructions (implemented in OS161)
- Atomic variables

Let's see how OS161 addresses this problem:

**Hardware Instructions** allow us to _test and modify_ the content of a word **_atomically_**. Conceptually what happens is:

        boolean test_and_set(boolean *test){
                boolean rv = *target;
                *target = true;
                return rv;
        }

but this is **implemented via hardware** and NOT via software, so that this execution is atomical. Solution:

        do{
                while(test_and_set(&lock)){;}     // wait for lock

                /* CRITICAL SECTION */

                lock = false;

        }while(true);

a variant of _test_and_set_ is _compare_and_swap_ which is general and not limited to the _true-false_ case, but works exactly the same.

This implementation only ensures the mutual exclusion, but there's no guarantee that each thread that needs a specific resource will eventually be able to acquire it. For this reason we introduce a **booking vector** _waiting[n]_ with _n = #threads_, in which each thread must set to 1 the corresponding flag whenever he intends to acquire the resource. This approach is called **Bounded waiting**.

Typically, _test_and_set_ or _compare_and_swap_ are not used with a standalone approach. Are in fact used as building blocks for other synchronization tools.

### MUTEX LOCKS

These are the simplest among the synchronization primitives. Mutexes are implemented via software and rely on a _lock_ (boolean variable) that can be released or acquired. Calls to **_acquire()_** and **_release()_** must therefore be atomic, and usually are implemented via hardware instructions such as _compare_and_swap_.

However, this approach involves a **busy wait** for the lock to be released (for this reason these are also called **spin locks** ), that can be acceptable in those applications where the wait time is typically very short, but is highly **inefficient** in scenarios where the waiting time is longer.

In OS161 we do have spinlocks that are implemented through a variant of _test_and_set_: the **_test-test-and-set_**. Its purpose is to address the main issue of busy waiting: high power consumption and high bus contention (since _test_and_set_ is an hardware component it can only be seen by one at a time). This optimized version includes a software 'test' (_get()_) in which we check whether lock has been released or not **before** the actual _test_and_set_.

Conceptually:

        spinlock_acquire(struct spinlock *splk){
                ...
                while(1){
                        if(spinlock_data_get(&splk->splk_lock) != 0){           // first get(). May be wrong,
                                continue;                                       // but is highly unlikely
                        }

                        if(spinlock_data_testandset(&splk->splk_lock) != 0){    // actual test_and_set
                                continue;
                        }
                        break;
                }
                ...
        }

REMEMBER: spinlocks must only be used in those scenarios in which we have **short critical sections**, therefore short waiting time.

### SEMAPHORES

Semaphores are a more general and powerful synchronization primitive than mutexes. A semaphore is essentially a counter that can be **decremented** when a resource is acquired and **incremented** when the resource is released. Unlike mutexes, semaphores can allow multiple threads to access a shared resource up to a certain limit.

There are two main types:

- **Counting semaphores**, which allow up to N threads to enter a critical section.
- **Binary semaphores**, which behave similarly to mutex locks (only one thread at a time).

#### Busy-Waiting Semaphores

The naive implementation of semaphores uses busy waiting in the _wait()_ operation (often called _P()_):

        wait(S) {
                while (S <= 0){;}       // busy wait
                S--;
        }

        signal(S) {
                S++;
        }

This solution is **inefficient**, as the CPU cycles are wasted in the spin loop. It is only acceptable when the expected waiting time is very short, similarly to spinlocks.

#### Non-Busy-Waiting Semaphores (OS161 Implementation)

To improve efficiency, OS161 implements non-busy waiting semaphores using **wait channels** and **spinlocks**. Instead of spinning, a thread is put to **sleep** if the semaphore’s count is zero, and **woken up** when another thread increments it via _V()_.

The semaphore structure in OS161 looks like this:

        struct semaphore {
                char *name;
                struct wchan *sem_wchan;     // wait channel to sleep/wake threads
                struct spinlock sem_lock;    // protects the structure
                volatile int count;          // number of available resources
        };

- _P()_ (**wait**): atomically checks the count and sleeps the thread if no resource is available.

        void P(struct semaphore *sem) {
                KASSERT(sem != NULL);
                KASSERT(curthread->t_in_interrupt == false); // cannot sleep in interrupts

                spinlock_acquire(&sem->sem_lock);

                while (sem->count == 0) {
                        wchan_sleep(sem->sem_wchan, &sem->sem_lock); // block until woken
                }

                KASSERT(sem->count > 0);
                sem->count--;

                spinlock_release(&sem->sem_lock);
        }

- _V()_ (**signal**): increments the count and wakes one thread waiting on the channel.

        void V(struct semaphore *sem) {
                KASSERT(sem != NULL);

                spinlock_acquire(&sem->sem_lock);

                sem->count++;
                KASSERT(sem->count > 0);
                wchan_wakeone(sem->sem_wchan, &sem->sem_lock); // wake one waiting thread

                spinlock_release(&sem->sem_lock);
        }

REMEMBER: semaphores must be used when you need to coordinate threads with **wait and signal behavior**.

### LOCKS - to be implemented in [LAB 3](#implementation)

A lock is similar to a binary semaphore with an initial value of 1. However, locks also enforce an additional constraint: **the thread that releases a lock must be the same thread that most recently acquired it**.

### CONDITION VARIABLES - to be implemented in [LAB 3](#implementation)

While mutexes and semaphores are powerful synchronization tools, they are often not enough when threads need to wait for **complex conditions** to become true. This is where condition variables come in.

A condition variable allows a thread to **sleep** while waiting for a condition to be true, and to be **woken up** when another thread signals that the condition might now be satisfied.

N.B. -> Condition variables are always used **in conjunction with a lock** to protect the shared data that represents the condition.

Conceptually:

- consumer:

        lock_acquire(buffer_lock);
        while (buffer_is_empty()) {
                cv_wait(buffer_cv, buffer_lock);  // sleep until data is available
        }
        consume_item();
        lock_release(buffer_lock);

- producer:

        lock_acquire(buffer_lock);
        produce_item();
        cv_signal(buffer_cv, buffer_lock);    // wake one waiting consumer
        lock_release(buffer_lock);

In OS161, condition variables are implemented with:

- a **wait channel** (_wchan_) for managing sleeping threads
- a **spinlock** for atomic access to the wait channel

The structure is defined in synch.h:

        struct cv {
        char *cv_name;
        struct wchan *cv_wchan;
        struct spinlock cv_lock;
        };

- _cv_wait()_

1.  Atomically releases the associated lock.
2.  Puts the thread to sleep on the CV’s wait channel.
3.  Re-acquires the lock when the thread wakes up.

        void cv_wait(struct cv *cv, struct lock *lock) {
        KASSERT(cv != NULL);
        KASSERT(lock != NULL);
        KASSERT(lock_do_i_hold(lock));

        spinlock_acquire(&cv->cv_lock);
        lock_release(lock);
        wchan_sleep(cv->cv_wchan, &cv->cv_lock);
        spinlock_release(&cv->cv_lock);
        lock_acquire(lock);
        }

- _cv_signal()_

        void cv_signal(struct cv *cv, struct lock *lock) {
        KASSERT(cv != NULL);
        KASSERT(lock != NULL);
        KASSERT(lock_do_i_hold(lock));

        spinlock_acquire(&cv->cv_lock);
        wchan_wakeone(cv->cv_wchan, &cv->cv_lock);      // wakes one thread waiting on the CV
        spinlock_release(&cv->cv_lock);
        }

- _cv_broadcast()_

        void cv_broadcast(struct cv *cv, struct lock *lock) {
        KASSERT(cv != NULL);
        KASSERT(lock != NULL);
        KASSERT(lock_do_i_hold(lock));

        spinlock_acquire(&cv->cv_lock);
        wchan_wakeall(cv->cv_wchan, &cv->cv_lock);      // wakes all
        spinlock_release(&cv->cv_lock);
        }

IMPORTANT:

- Condition variables provide no mutual exclusion themselves: the lock you use with them must protect the condition.
- Always use _while_, not _if_, when checking the condition before sleeping. Conditions may change between wake-up and re-acquiring the lock.

Usage pattern:

        lock_acquire(lock);
        while (!condition) {
        cv_wait(cond_var, lock);
        }
        do_something();
        lock_release(lock);

### WAIT CHANNELS (long story short)

A **wait channel** (or wchan) is a low-level synchronization mechanism used in the OS161 kernel to **block** and **wake** threads. Unlike condition variables or semaphores, wait channels are **not exposed directly to the application programmer**. Instead, they are used internally to implement higher-level constructs like semaphores and condition variables.

A wait channel allows threads to:

- Sleep (block) while waiting for a specific event to occur.
- Wake up when the event happens.

They work in conjunction with _spinlocks_, ensuring that sleeping and waking operations are atomic and thread-safe.

## IMPLEMENTATION

### LOCK

In this lab activity we're requested to implement our own version of locks for the os161 kernel. There are two different ways to address this request:

- by using os161 pre-implemented **semaphores**, which internally:
  - protects the access to the counter by mean of a _spinlock_
  - sleeps threads by mean of a _wchan_
- by building our lock from scratch, by means of **waiting channels** and **spin locks**

Here are reported the most significant differences between these two approaches.

|                           |                 Semaphore                  |       wchan + spinlock       |
| ------------------------- | :----------------------------------------: | :--------------------------: |
| **Abstraction**           |                    High                    |      Low (more control)      |
| **Complexity**            |                   Lower                    |            Higher            |
| **Flexibility**           |       Limited by semaphores behavior       | Programmes has total control |
| **Internal dependencies** | relies on semaphores and their correctness |          Autonomous          |

#### STEP 1 - configurations

Firstly, we must create a new configuration in _kern/conf/conf.kern_:

        defoption synch

**NB**: here we DO NOT need to add any _optfile ..._.

Details on how to create a new config : [LAB 1](#lab-1---options)

#### STEP 2 - _synch.h_

Once we've created our configuration successfully, we can implement our **lock**.
In _kern/include/synch.h_ we want to **overwrite** the naive implementation of a lock that is already present, like so:

        struct lock {
                char *lk_name;

        #if OPT_SYNCH

                #if USE_SEMAPHORE_FOR_LOCK
                        struct semaphore *lk_sem;
                #else
                        struct wchan *lk_wchan;
                #endif
                        struct spinlock lk_lock;
                        volatile struct thread *lk_owner;
        #endif
        };

As you may notice, here we include a _struct spinlock_ regardless whether _USE_SEMAPHORE_FOR_LOCK_ equals _1_ or not. That's not exactly what you'd expect from what we said before. In fact _struct semaphore_ already has a spinlock inside, so why do we need another one here?

        // semaphore struct implementation

        struct semaphore {
                char *sem_name;
                struct wchan *sem_wchan;
                struct spinlock sem_lock;
                volatile unsigned sem_count;
        };

Semaphores use spinlocks so that \*_sem_wchan_ and _sem_count_ are protected, therefore operations like _P()_ and _V()_ are atomical.

But, in our _lock_ we also need to protect **_volatile struct thread \*lk_owner_**, and for this reason a _spinlock_ is required anyway.

#### STEP 3 - _synch.c_

Now we need to write the implementation for these methods:

        struct lock *lock_create(const char *name);
        void lock_destroy(struct lock *);
        void lock_acquire(struct lock *);
        void lock_release(struct lock *);
        bool lock_do_i_hold(struct lock *);

These must be **atomic**.

Follows the content related to lock implementation in _synch.c_.

        ////////////////////////////////////////////////////////////
        //
        // Lock.

        struct lock *
        lock_create(const char *name)
        {
                struct lock *lock;

                lock = kmalloc(sizeof(*lock));
                if (lock == NULL) {
                        return NULL;
                }

                lock->lk_name = kstrdup(name);
                if (lock->lk_name == NULL) {
                        kfree(lock);
                        return NULL;
                }

        #if OPT_SYNCH

                #if USE_SEMAPHORE_FOR_LOCK
                        lock->lk_sem = sem_create(lock->lk_name, 1);
                        if (lock->lk_sem == NULL) {
                                kfree(lock->lk_name);
                                kfree(lock);
                                return NULL;
                        }
                #else
                        lock->lk_wchan = wchan_create(lock->lk_name);
                        if (lock->lk_wchan == NULL) {
                                kfree(lock->lk_name);
                                kfree(lock);
                                return NULL;
                        }
                        lock->lk_owner = NULL;
                        spinlock_init(&lock->lk_lock);
                #endif

        #endif  // OPT_SYNCH

                return lock;
        }


        void
        lock_destroy(struct lock *lock)
        {
                KASSERT(lock != NULL);

        #if OPT_SYNCH
                spinlock_cleanup(&lock->lk_lock);
                #if USE_SEMAPHORE_FOR_LOCK
                        sem_destroy(lock->lk_sem);
                #else
                        wchan_destroy(lock->lk_wchan);
                #endif
        #endif
                kfree(lock->lk_name);
                kfree(lock);
        }

        void
        lock_acquire(struct lock *lock)
        {
        #if OPT_SYNCH

                KASSERT(lock != NULL);

                if (lock_do_i_hold(lock)) {
                kprintf("AAACKK!\n");
                }

                KASSERT(!(lock_do_i_hold(lock)));

                KASSERT(curthread->t_in_interrupt == false);

                #if USE_SEMAPHORE_FOR_LOCK

                        P(lock->lk_sem);
                        spinlock_acquire(&lock->lk_lock);
                #else
                        spinlock_acquire(&lock->lk_lock);
                        while (lock->lk_owner != NULL) {
                                wchan_sleep(lock->lk_wchan, &lock->lk_lock);
                        }
                #endif

                KASSERT(lock->lk_owner == NULL);
                lock->lk_owner=curthread;
                spinlock_release(&lock->lk_lock);
        #endif
                (void)lock;  // suppress warning until code gets written
        }

        void
        lock_release(struct lock *lock)
        {
        #if OPT_SYNCH

                KASSERT(lock != NULL);
                KASSERT(lock_do_i_hold(lock));
                spinlock_acquire(&lock->lk_lock);
                lock->lk_owner=NULL;

                #if USE_SEMAPHORE_FOR_LOCK
                        V(lock->lk_sem);
                #else
                        wchan_wakeone(lock->lk_wchan, &lock->lk_lock);
                #endif

                spinlock_release(&lock->lk_lock);
        #endif

                (void)lock;  // suppress warning until code gets written
        }

        bool
        lock_do_i_hold(struct lock *lock)
        {
                // Write this
        #if OPT_SYNCH
                bool res;
                spinlock_acquire(&lock->lk_lock);
                res = lock->lk_owner == curthread;
                spinlock_release(&lock->lk_lock);
                return res;
        #endif

                (void)lock;  // suppress warning until code gets written

                return true; // dummy until code gets written
        }

**_Relevant notes_**:

- in _synch.h_ you must define **_USE_SEMAPHORE_FOR_LOCK_** and set it according to whether you want to use the _naive_ implementation with semaphores or the more 'accurate' one.
- in _lock_acquire()_, while using the "semaphore approach", you MUST call _P(lock->lk_sem)_ **before** calling _spinlock_acquire(&lock->lk_lock)_, otherwise:
  - _P()_ may block if the semaphore count is 0.
  - But spinlocks must not block—they are designed to spin and must be released quickly.
  - Blocking while holding a spinlock would prevent the thread currently holding the semaphore from progressing (and releasing it), resulting in a **deadlock**.
- differently, in _lock_release()_ it's actually safe to call _V(lock->lk_sem)_ while holding a spinlock. This happens because _V()_ simply increments the semaphore count and wakes up a waiting thread, if any. Therefore it actually never blocks, differently from what _P()_ does.

### CONDITION VARIABLE

In a dual way, we can now implement our **condition variables**.

#### STEP 1 - configurations

Here we only need the same configuration that we should already have implemented for locks. ([v. lock's STEP 1](#step-1---configurations))

#### STEP 2 - _synch.h_

        struct cv {
                char *cv_name;

        #if OPT_SYNCH
                struct wchan *cv_wchan;
                struct spinlock cv_lock;
        #endif
        };

#### STEP 3 - _synch.c_

        struct cv *
        cv_create(const char *name)
        {
                struct cv *cv;

                cv = kmalloc(sizeof(*cv));
                if (cv == NULL) {
                        return NULL;
                }

                cv->cv_name = kstrdup(name);
                if (cv->cv_name==NULL) {
                        kfree(cv);
                        return NULL;
                }

        #if OPT_SYNCH
                cv->cv_wchan = wchan_create(cv->cv_name);
                if (cv->cv_wchan == NULL) {
                        kfree(cv->cv_name);
                        kfree(cv);
                        return NULL;
                }
                spinlock_init(&cv->cv_lock);
        #endif
                return cv;
        }

        void
        cv_destroy(struct cv *cv)
        {
                KASSERT(cv != NULL);

        #if OPT_SYNCH
                spinlock_cleanup(&cv->cv_lock);
                wchan_destroy(cv->cv_wchan);
        #endif
                kfree(cv->cv_name);
                kfree(cv);
        }

        void
        cv_wait(struct cv *cv, struct lock *lock)
        {
        #if OPT_SYNCH
                KASSERT(lock != NULL);
                KASSERT(cv != NULL);
                KASSERT(lock_do_i_hold(lock));

                spinlock_acquire(&cv->cv_lock);
                lock_release(lock);
                wchan_sleep(cv->cv_wchan,&cv->cv_lock);
                spinlock_release(&cv->cv_lock);
                lock_acquire(lock);
        #endif

                (void)cv;    // suppress warning until code gets written
                (void)lock;  // suppress warning until code gets written
        }

        void
        cv_signal(struct cv *cv, struct lock *lock)
        {
        #if OPT_SYNCH
                KASSERT(lock != NULL);
                KASSERT(cv != NULL);
                KASSERT(lock_do_i_hold(lock));

                spinlock_acquire(&cv->cv_lock);
                wchan_wakeone(cv->cv_wchan,&cv->cv_lock);
                spinlock_release(&cv->cv_lock);
        #endif
                (void)cv;    // suppress warning until code gets written
                (void)lock;  // suppress warning until code gets written
        }

        void
        cv_broadcast(struct cv *cv, struct lock *lock)
        {
        #if OPT_SYNCH
                KASSERT(lock != NULL);
                KASSERT(cv != NULL);
                KASSERT(lock_do_i_hold(lock));

                spinlock_acquire(&cv->cv_lock);
                wchan_wakeall(cv->cv_wchan,&cv->cv_lock);
                spinlock_release(&cv->cv_lock);
        #endif
                (void)cv;    // suppress warning until code gets written
                (void)lock;  // suppress warning until code gets written
        }
