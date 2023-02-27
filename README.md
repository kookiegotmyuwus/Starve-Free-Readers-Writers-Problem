# Starve Free Readers-Writers Problem

Multiple processes that run in a computer may need to access the same resources at the same time. Take a file, for example. Some processes would **write** to that file, and others would **read** from that file. The file's contents are updated when the process writes to it, not when some process reads from it. Hence, it is okay for two processes to read from it simultaneously but not for any other process to access a resource when a process is writing to it.

> We must devise an algorithm that will allow simultaneous reading, but mutually exclusive writing to a resource.

This class of problems is termed as the Readers-Writers problems.

## Table of Contents

1. [Semaphores and Mutex Locks](#alpha)
2. [First Readers-Writers Problem](#beta)
3. [Second Readers-Writers Problem](#gamma)
4. [Third (Starve-Free) Readers-Writers Problem](#delta)

## <a name="alpha"></a>Semaphores and Mutex Locks

Semaphores and mutex locks are data structures that control access of resources by processes.

### Mutex Lock

A process can `acquire` a mutex, which grants it access to a resource. This lock can be acquired when it is available. When the lock is not available, the process is kept in busy waiting, until it becomes available. Once it finishes accessing a resource, it will `release` it.

```cpp
class MutexLock {
    bool available;

    // constructor
    MutexLock () {
        available = true; // The resource is initially available.
    }

    acquire() {
        while (!available); // busy waiting

        // when it available, claim resource and set available to false
        available = false;
    }

    // Once the process is done with the resource, it releases it for other processes to use.
    release () {
        available = true;
    }
}
```

### Semaphore

A semaphore is essentially an integer value. If it is positive, it represents the number of resources corresponding to that semaphore available. If it is negative, it represents the number of processes in queue for that semaphore.

To prevent busy waiting like in mutex locks, a process that is waiting for a resource is blocked and pushed to the waiting queue of the semaphore. Once it is ready to be executed again, it is popped from the waiting list, awakened, and pushed to the ready queue.

The data structure for the semaphore can be represented by this class:

```cpp
class Semaphore {
    int value; // value of semaphore
    queue waitlist; // list of waiting process PCBs

    // constructor
    Semaphore () {
        this.value = 1; // set initial value of semaphore to 1
        queue<PCB> waitlist; // initialise an empty queue to store PCB structs
    }

    wait (PCB &p) {
        this.value--;
        if (this.value < 0) { // Adds another processes to the waiting list
            this.waitlist.push(p);
            block(p); // System call
        }
    }

    signal () {
        this.value++; // release a resource
        if (this.value <= 0) { // means at least one process is still waiting
            p = this.waitlist.pop();
            wakeup(p); // System call
        }
    }
}
```

## <a name="beta"></a>First Readers-Writers Problem

The first problem focuses on allowing multiple readers to read the same resource at the same time, while still maintaining mutual exclusion between writers. Since this approach favours readers over writers, this is known as the **_readers-preference_** problem.

The pseudo-code to solve this problem is shown below: (explanation of the code is given in the comments).

#### Global variables

```cpp
ResourceSemaphore = Semaphore(); // Semaphore to control resource access
ReadMutex = MutexLock(); // Mutex lock to control critical section when updating reader_count
int reader_count = 0; // Number of readers accessing resource at the moment.
```

#### Implementation for writers

```cpp
writer () {
    // Wait till all readers release the resource, then lock and take control of it
    ResourceSemaphore.wait();
    /* Critical Section (Process writes to resource) */
    ResourceSemaphore.signal(); // Release resource
}
```

#### Implementation for readers

```cpp
reader () {
    /* BEGIN ENTRY SECTION */
    // Only one process must be allowed to update the reader count at a time.
    ReadMutex.acquire();

    /* BEGIN CRITICAL SECTION */
    // This is a critical section because reader_count is a global variable and is being updated.

    // Add 1 to it because a new process has come to read it.
    reader_count++;

    // Checks if this process is the first to read it after a write operation has been done.
    if (reader_count == 1)
        // If we are the first reader, then lock the resource from all writers (but not other readers).
        ResourceSemaphore.wait();

    ReadMutex.release(); // Done updating global variable reader_count
    /* END CRITICAL SECTION */

    /* END ENTRY SECTION */

    // Once the access to the resource is granted, we read it
    /*
        READING SECTION
    */


    /* BEGIN EXIT SECTION */
    // Only one process must be allowded to update the reader count at a time.
    ReadMutex.acquire();

    /* BEGIN CRITICAL SECTION */
    // This is a critical section because reader_count is a global variable and is being updated.

    // Subtract 1 from it because a process has finished reading it.
    reader_count--;

    // Checks if this process is the first to read it after a write operation has been done.
    if (reader_count == 0)
        // If we are the last reader, then release the lock from the resource, and
        // allow writers to write to it if they want.
        ResourceSemaphore.signal();

    ReadMutex.release(); // Done updating global variable reader_count
    /* END CRITICAL SECTION */

    /* END EXIT SECTION */
}
```

In the above code, writers must wait till **all waiting readers** are done reading the resources, before even a single writer can gain access to it. Hence, this method favours readers, and may lead to **starvation of writers**.

## <a name="gamma"></a>Second Readers-Writers Problem

As mentioned above, the code for the first problem allowed multiple readers, however, allowed writers to starve. The second solution helps prevent starvation of writers and is called the **writer-preference** problem.

The pseudo-code to solve this problem is shown below: (explanation of the code is given in the comments).

#### Global variables

```cpp
// Semaphore to control resource access
ResourceSemaphore = Semaphore()
// Semaphore to prevent readers from accessing resource if any writers are waiting
ReadTrySemaphore = Semaphore()

ReadMutex = MutexLock() // Mutex lock to control critical section when updating reader_count
WriteMutex = MutexLock() // Mutex lock to control critical section when updating writer_count
int reader_count = 0; // Number of readers currently access the resource
int writer_count = 0; // Number of writers currently accessing and waiting to access the resource
```

#### Implementation for writers

```cpp
writer () {
    /* BEGIN ENTRY SECTION */
    // Only one process must be allowed to update the writer count at a time.
    WriteMutex.acquire();

    /* BEGIN CRITICAL SECTION */
    // Add 1 to it because a new process has come to write to it.
    writer_count++;
    // Check if this process is the first to write to it after a read operation has been done.
    if (writer_count == 1)
        // If we start writing to it, lock other readers out until all writes are done
        ReadTrySemaphore.wait();


    // Done updating the writer_count variable (leave critical section)
    WriteMutex.release();
    /* END CRITICAL SECTION */

    // Reserve resource for self, not allowing other writers to write to it.
    ResourceSemaphore.wait();
    /* END ENTRY SECTION */

    // Once the access to the resource is granted, we write to it
    /*
        WRITING SECTION
    */
    // Release file from lock
    ResourceSemaphore.signal();


    /* BEGIN EXIT SECTION */
    // Only one process must be allowded to update the writer count at a time.
    WriteMutex.acquire();

    /* BEGIN CRITICAL SECTION */
    // This is a critical section because writer_count is a global variable and is being updated.

    // Subtract 1 from it because a process has finished reading it.
    writer_count--;

    // Checks if this process is the last to write to it.
    if (writer_count == 0)
        // If we are the last writer, then release the lock from the resource, and
        // allow readers to read it if they want.
        ReadTrySemaphore.signal();

    WriteMutex.release(); // Done updating the writer_count variable (leave critical section)
    /* END CRITICAL SECTION */

    /* END EXIT SECTION */
}
```

#### Implementation for readers

```cpp
reader () {
    // Reader must wait till all writers are done writing
    ReadTrySemaphore.wait();

    /* BEGIN ENTRY SECTION */

    // Only one process must be allowed to update the reader count at a time.
    ReadMutex.acquire();

    // Add 1 to it because a new process has come to read it.
    reader_count++;
    // Check if this process is the first to read it after a write operation has been done.
    if (reader_count == 1)
        // If we are the first reader, then lock the resource from all writers (but not other readers).
        ResourceSemaphore.wait();

    ReadMutex.release(); // Done updating reader_count
    /* END CRITICAL SECTION */

    ReadTrySemaphore.signal(); // Done trying to access resource.

    // Once the access to the resource is granted, we read it
    /*
        READING SECTION
    */

    /* BEGIN EXIT SECTION */
    // Only one process must be allowded to update the reader count at a time.
    ReadMutex.acquire();

    /* BEGIN CRITICAL SECTION */
    // This is a critical section because reader_count is a global variable and is being updated.

    // Subtract 1 from it because a process has finished reading it.
    reader_count--;

    // Checks if this process is the last to read it.
    if (reader_count == 0)
        // If we are the last reader, then release the lock from the resource, and
        // allow writers to write to it if they want.
        ResourceSemaphore.signal();
    /* END CRITICAL SECTION */

    ReadMutex.release();           // Release read mutex
    /* END EXIT SECTION */
}
```

Although this solution prevents writers from starvation, it allows readers to starve.

## <a name="delta"></a> Third Readers-Writers Problem (Starve-Free!)

As both of the above solutions allow either readers or writers to starve, we need a solution that prevents starvation altogether.

This calls for a mehtod that puts a bound on the waiting time for all processes.

The starve-free solution is described with the help of the following code:

#### Global variables

```cpp
// Semaphore to control resource access
ResourceSemaphore = Semaphore()
// Semaphore to preserve ordering of requests
ServiceQueueSemaphore = Semaphore()

ReadMutex = MutexLock() // Mutex lock to control critical section when updating reader_count
int reader_count = 0; // Number of readers currently access the resource
```

#### Implementation for writers

```cpp
writer () {
    /* BEGIN ENTRY SECTION */

    // Wait for earlier process to finish accessing resource
    ServiceQueueSemaphore.wait();
    // Request exclusive access to resource for writing
    ResourceSemaphore.wait();
    // Once access is granted, allow next item in queue to be serviced
    ServiceQueueSemaphore.signal();

    /* END ENTRY SECTION */

    // Once the access to the resource is granted, we write to it
    /*
        WRITING SECTION
    */

    /* BEGIN EXIT SECTION */
    // Release file from lock after done accessing
    ResourceSemaphore.signal();
    /* END EXIT SECTION */
}
```

#### Implementation for readers

```cpp
reader () {
	/* BEGIN ENTRY SECTION */
	// Wait for earlier process to finish accessing resource
	ServiceQueueSemaphore.wait();

	// Only one process must be allowed to update the reader count at a time.
	ReadMutex.acquire();

	// Add 1 to it because a new process has come to read it.
	reader_count++;
	// Check if this process is the first to read it after a write operation has been done.
    if (reader_count == 1)
	    // If we are the first reader, then lock the resource from all writers (but not other readers).
        ResourceSemaphore.wait();

	// Once access is granted, allow next item in queue to be serviced
	ServiceQueueSemaphore.signal();

	ReadMutex.release(); // Done updating reader_count
	/* END CRITICAL SECTION */

	// Once the access to the resource is granted, we read it
    /*
	    READING SECTION
	*/

	/* BEGIN EXIT SECTION */
	// Only one process must be allowded to update the reader count at a time.
    ReadMutex.acquire();

	/* BEGIN CRITICAL SECTION */
    // This is a critical section because reader_count is a global variable and is being updated.

    // Subtract 1 from it because a process has finished reading it.
	reader_count--;

	// Checks if this process is the last to read it.
    if (reader_count == 0)
	    // If we are the last reader, then release the lock from the resource, and
	    // allow writers to write to it if they want.
        ResourceSemaphore.signal();
	/* END CRITICAL SECTION */

    ReadMutex.release();           // Release read mutex
   	/* END EXIT SECTION */
}
```

This method prevents starvation of both readers and writers because it follows the following protocol:

- If a reader is reading a resource:
  - Allow all subsequent readers to read as well.
  - Any subsequent writer is pushed to the waiting queue. The writer is given access once all readers are finished reading.
- If a writer is writing to a resource:
  - Push any subsequent readers / writers to the waiting queue.

Additional points:

- As a queue follows a FIFO ordering, waiting time for all processes is bound. No process that comes later is given priority over the processes already waiting in queue based on their type of action (read/write).
- A reader_count is maintained for Readers to allow for simultaneous reading of a resource. Such a value is not required for writers because we know that writer_count will always be either 0 or 1.