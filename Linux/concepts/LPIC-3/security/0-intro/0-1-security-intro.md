# Security Fundamentals

> **Scope:** Supplementary introductory material. This section provides foundational security knowledge; it is not a standalone LPIC-3 303 exam objective.

This section introduces foundational information-security concepts, common software vulnerabilities, memory-corruption bugs, and Linux security updates.

## What Is Information Security?

Information security (InfoSec) protects information and the systems that store, process, and transmit it. A common foundational model is the **CIA triad**, consisting of three objectives:

```
                                  /\
                                 /  \
                                /    \
                               /      \
        Confidentiality (C)   /        \  Integrity (I)
                             /          \ 
                            /            \
                           /______________\
                           Availability (A)
```

| Objective | Meaning | Example |
|---|---|---|
| **Confidentiality** | Preventing unauthorized disclosure of information. | Restricting access to private keys and sensitive files. |
| **Integrity** | Preventing or detecting unauthorized or unintended modification or destruction of information. | Verifying package signatures and file checksums. |
| **Availability** | Ensuring authorized users can access systems and information when required. | Redundant services, backups, and protection against denial-of-service attacks. |

These objectives can conflict. For example, restricting access too aggressively may protect confidentiality but prevent legitimate users from accessing a service.

- ### Confidentiality and Cryptography

    **Confidentiality** prevents unauthorized disclosure of information. Cryptography, particularly encryption, is a major mechanism for protecting confidentiality, but other controls are also necessary.

    - **Encryption:** Protects data from being read by parties without the appropriate decryption key.
    - **Access control:** Restricts access to authorized users and processes.
    - **Authentication:** Helps establish identity before access is granted.

    Encryption does not protect data if keys are compromised or sensitive information is exposed before encryption or after decryption.

- ### Data Integrity and Data Origin Authentication

    - **Data integrity** means protecting the accuracy and completeness of data against unauthorized or unintended modification, insertion, deletion, or corruption.

        If system X sends data to system Y, integrity mechanisms help Y detect whether the data has been altered in transit or otherwise tampered with.

        **Common mechanisms include:**

        - **Cryptographic hash:** Produces a digest that can help detect changes when compared with a trusted reference. A hash alone does not authenticate the source.
        - **Message Authentication Code (MAC):** Uses a shared secret key to provide message integrity and authentication to parties possessing that key.
        - **Digital signature:** Allows verification of signed data integrity and that the signature corresponds to a particular signing key.

    - **source integrity**
        - **Data origin authentication** provides assurance that data came from the claimed source. It is distinct from **entity authentication**, which establishes the identity of a communicating user or system.

        - **Non-repudiation** concerns evidence that can help prevent a party from credibly denying an action, such as signing a message. Digital signatures can support non-repudiation, but a valid signature alone does not prove the real-world identity of the signer or automatically establish legal non-repudiation. Identity verification, key ownership, key protection, and appropriate procedures also matter.

### Relative Importance of the CIA Triad

**Confidentiality**, **integrity**, and **availability** are all fundamental security objectives, but their relative importance varies according to an organization's mission, business requirements, threat model, and risk assessment.

For example:

- **Intelligence systems:** Unauthorized disclosure may have exceptionally serious consequences, making confidentiality a major priority.
- **Financial systems:** Integrity of transactions and confidentiality of customer information are critical; availability is also essential.
- **Public-facing services:** Availability may be a major priority, while confidentiality and integrity remain necessary.

These are illustrative examples, not universal rankings. Even when one objective receives greater emphasis, neglecting the others can create unacceptable risks.

**Threat modeling** identifies assets, potential attackers, attack paths, and threats. **Risk assessment** evaluates the likelihood and impact of those threats in the presence of vulnerabilities and existing controls. Risk management uses this information to select and prioritize security measures.

---

## Keeping Linux Systems Updated

Software vulnerabilities are weaknesses that can be exploited to compromise confidentiality, integrity, availability, or other security properties. Vulnerabilities may exist in the Linux kernel, user-space applications, libraries, network services, and other software components.

### CVE

**CVE (Common Vulnerabilities and Exposures)** is a system for identifying publicly disclosed cybersecurity vulnerabilities using standardized identifiers.

A CVE identifier has the format `CVE-YYYY-NNNNN`, although the numeric suffix may contain more than five digits.

The CVE program helps security professionals reference the same vulnerability consistently across advisories, scanners, and security tools.

**Resources:**

- [Official CVE program](https://www.cve.org/)
- [MITRE CVE search for Linux-related vulnerabilities](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=linux)

**Important distinctions:**

- A CVE identifier identifies a vulnerability; it does not itself fix or detect it.
- A vulnerability affecting Linux software may affect only particular products, versions, configurations, or architectures.
- Not every security issue has a CVE identifier.
- A system can remain vulnerable even when its package version appears older or newer than a version mentioned in an advisory. Distributions may backport security fixes without adopting the upstream version number.

> **Remote Code Execution (RCE)** is a vulnerability that allows an attacker to execute arbitrary code on a target system remotely, potentially with the privileges of the vulnerable service or process.

### Security Updates with DNF

On Fedora, RHEL, Rocky Linux, AlmaLinux, and other DNF-based systems, DNF manages packages and can use repository-provided security advisory metadata.

- `dnf check-update` → Checks for available package updates. Exit status `100` normally indicates that updates are available; `0` means none were found, and other nonzero statuses can indicate errors.
- `dnf updateinfo` → Displays available update-advisory information when supported by the repository metadata.
- `dnf updateinfo list --security` → Lists security-related advisories or updates known through the configured repositories.
- `dnf update --security` → Applies applicable security-related package updates identified by the available advisory metadata.
- `dnf update` → Updates installed packages for which newer applicable versions are available.

**Important:** Security-filtered updates depend on correctly maintained repository advisory metadata. They do not guarantee that every vulnerability on the machine will be identified or fixed. Regular full updates, appropriate repository configuration, and vulnerability assessment remain important.

For production systems, test updates where appropriate, plan for service restarts or reboots, and verify that the expected package versions and security fixes are installed.

---

## Memory Corruption

**Memory corruption** occurs when a program incorrectly accesses or modifies memory, potentially violating the intended structure or contents of its address space. It often results from programming errors such as out-of-bounds access, use-after-free, or incorrect pointer handling.

Memory-corruption vulnerabilities can cause:

- Application crashes or unexpected behavior.
- Disclosure of sensitive data stored in memory.
- Modification of program state or control flow.
- Arbitrary code execution under the privileges of the vulnerable process, in some cases.

> An **address space** is the range of memory addresses that a process can use to access its code, data, stack, heap, shared libraries, and other mapped memory regions.
>
> On Linux, each process typically has its own virtual address space, which the operating system and CPU's memory-management hardware map to physical memory or other backing storage.
>
> *Example:* ASLR randomizes the locations of certain regions within a process's virtual address space, making it harder for an attacker to predict memory addresses needed to exploit a vulnerability.

### Common Memory-Corruption Bugs

| Bug class | Description |
|---|---|
| *Buffer overflow* | Writing beyond the allocated bounds of a buffer. |
| *Out-of-bounds read* | Reading outside the valid bounds of an object or buffer, potentially exposing data. |
| *Use-after-free* | Accessing memory through a pointer after the associated allocation has been freed. |
| *Double-free* | Releasing the same memory allocation more than once, potentially corrupting allocator state. |
| *Integer overflow* | An arithmetic operation produces a value outside the representable range, potentially causing incorrect size calculations or bounds checks. |
| *Stack exhaustion* | Excessive stack consumption, often through deep recursion, causes the process to fail. This is distinct from a stack-based buffer overflow. |

Memory corruption is particularly associated with languages that allow direct memory manipulation, such as `C` and `C++`. However, it is not exclusive to these languages.

> Forgetting to free dynamically allocated memory causes a **memory leak**. It can consume increasing amounts of memory, potentially causing performance degradation or denial of service (DoS) if an attacker can repeatedly trigger the leak. In a long-running process, severe leaks may eventually exhaust available memory.

> ### Buffer
> A buffer is a temporary area of memory used to hold data while it is waiting to be processed, transferred, or consumed.
>
> Think of it like a waiting area: 
> 1. data arrives
> 2. waits there for a while 
> 3. and is then read or processed.
>
> Examples in Linux
> - Buffer in a program: A program receives network data and stores it in memory before processing it.
> - Kernel ring buffer: The Linux kernel stores log messages in a circular memory buffer. When you run **`dmesg`**, you read messages from this buffer.
> - I/O buffer: Data is temporarily stored while being read from or written to a device.
>
> **Why is it called a ring buffer?**
> 
> A ring buffer uses a fixed-size memory area as if its beginning and end were connected in a circle. When the write position reaches the end, it wraps around to the beginning, potentially overwriting older data if the buffer is full.
>
> **Remember:** A buffer is the general concept; a ring buffer is one specific type of buffer. A buffer overflow occurs when a program writes beyond the space available in a buffer.

> ## Static memory allocation vs Dynamic memory allocation
> The main difference is **when memory is allocated**, **how its size is determined**, and **how long it remains allocated**.
> - ### Static memory allocation
>   Memory is reserved before the program runs, or as part of a function's fixed-size stack frame when the function is entered.
>   ```c
>   int numbers[10];
>   ```
> 
>   Here, numbers has space for 10 integers, and its size is fixed.
>   - The size is fixed for this array.
>   - If it is a local variable, it typically exists while the function is executing.
>   - If it has static storage duration, it exists for the lifetime of the program.
>
> - ### Dynamic memory allocation
>   Memory is requested while the program is running, allowing the program to choose the required size at runtime.
>
>   ```c
>   int *numbers = malloc(10 * sizeof(int));
>   
>   if (numbers != NULL) {
>       /* Use numbers */
>       free(numbers);
>   }
>   ```
> 
>   - The size can be determined at runtime.
>   - The allocated memory remains allocated until it is released with free() or the process terminates.
>   - The programmer must manage its lifetime correctly in C.
>
> ### Comparison
>
> | Feature | Static allocation | Dynamic allocation |
> | --- | --- | --- |
> | When | Before execution, or at a function's entry for fixed local storage | During execution
> | Size | Fixed for the object | Can be chosen at runtime
> | Typical location | Static storage or stack | Heap
> | Lifetime | Depends on storage duration | Until released or process termination
> | Management | Usually automatic | Explicit allocation and release in C
>
> **Important distinction:** Static allocation does not always mean memory is stored in the static data segment. Local fixed-size arrays are typically stored on the stack, whereas global variables and variables declared with static have static storage duration.
>
> **Remember:** Static allocation is suitable when storage requirements and lifetimes are known; dynamic allocation is useful when memory requirements need to be decided at runtime.
> 



### Dynamic Memory Allocation in C

`C` programs can request dynamic memory using functions such as `malloc()` and `calloc()`:

- `malloc(size)` allocates a block of memory but leaves its contents uninitialized.
- `calloc(count, size)` allocates memory for an array and initializes the allocated bytes to zero.
- `free(ptr)` releases a dynamically allocated block.

These functions do not automatically protect a program from invalid memory accesses. The programmer must correctly manage allocation sizes, object lifetimes, pointers, and bounds.

Memory-safe languages such as `Java`, `Python`, and `C#` generally prevent many classes of memory corruption through language or runtime checks. However, they are not immune to every memory-safety problem, particularly when native extensions, unsafe operations, or vulnerable runtime components are involved.

---

## Buffer Overflow

A **buffer overflow** occurs when a program reads or writes data beyond the valid bounds of a buffer. A buffer is a region of memory used to hold data temporarily while a program processes or transfers it.

For example, suppose a program allocates a 100-byte character array but copies 200 bytes into it without checking the destination capacity. The extra data may overwrite adjacent memory, resulting in a crash or, under certain conditions, an exploitable vulnerability.

The actual overflow depends on the number of bytes written and the available capacity - not simply on the number of characters entered by a user.

### Stack-Based Buffer Overflow Example

Consider this C program, saved as `buf.c`:

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[])
{
    char buf[100];

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <input>\\n", argv[0]);
        return 1;
    }

    strcpy(buf, argv[1]);
    printf("Input was: %s\\n", buf);

    return 0;
}
```

The program allocates a 100-byte array named `buf` and copies the command-line argument into it using `strcpy()`.

The problem is that `strcpy()` does not know the destination buffer's capacity. It copies the source string, including its terminating null byte, without checking whether the destination is large enough.

Consequently, a sufficiently long input causes a **stack-based buffer overflow** because `buf` is a local variable typically stored on the stack.

Compile the program for a basic demonstration:

```bash
gcc -Wall -Wextra -o buf buf.c
```

Test it with a short input:

```bash
./buf hello
```

Then supply a longer input:

```bash
./buf "$(python3 -c 'print("A" * 200)')"
```

The second invocation attempts to copy 200 `A` characters plus the terminating null byte into a 100-byte buffer. This invokes undefined behavior: the program may crash, corrupt memory, or appear to work despite the bug.

**Do not rely on a crash as proof that the vulnerability exists or that it has been fixed.**

The example is intentionally vulnerable and should be used only in an isolated lab. Do not run untrusted exploit inputs against production systems.

### Compiler and Operating-System Protections

The instructor's original compilation command included these options:

| Option | Effect |
|---|---|
| `-fno-stack-protector` | Disables compiler-generated stack-protector checks for applicable functions. |
| `-z execstack` | Marks the executable as requesting an executable stack, weakening a common memory-execution protection. |

These options are not required to demonstrate that the program contains a buffer overflow. They deliberately weaken security and should not be used for ordinary application builds.

Modern systems can apply several complementary defenses:

- **Stack canaries:** Detect certain stack overwrites before a protected function returns.
- **NX / DEP:** Prevent execution from memory regions that are not marked executable, where supported.
- **ASLR:** Randomizes relevant memory locations to make address prediction harder.
- **PIE (Position-Independent Executable):** Allows the main executable's code location to be randomized when combined with appropriate loader and kernel support.
- **Bounds checking and safer programming practices:** Prevent the invalid write in the first place.

These protections reduce exploitability; they do not eliminate the underlying programming error.

The preferred fix is to correct the code - for example, validate input length and use an appropriate bounded operation with correct handling of the destination capacity. Simply replacing `strcpy()` with another function does not automatically make code safe if its bounds and termination rules are handled incorrectly.

---

## Address Space Layout Randomization (ASLR)

**Address Space Layout Randomization (ASLR)** is a security mitigation that randomizes the locations of relevant memory regions, making it more difficult for an attacker to predict addresses needed for exploitation.

Depending on the architecture, kernel configuration, executable format, and system configuration, ASLR can randomize locations associated with:

- The stack.
- The memory-mapped region and shared libraries.
- The virtual dynamic shared object (VDSO).
- The heap.
- The main executable, when suitable PIE support is present.

ASLR does not prevent memory corruption. It makes certain exploitation techniques less reliable, particularly when an attacker needs to predict memory addresses. Its effectiveness can be reduced by information leaks or other weaknesses.

### Configuring ASLR on Linux

The kernel exposes the system-wide ASLR setting through:

```bash
cat /proc/sys/kernel/randomize_va_space
```

The commonly supported values are:

| Value | Meaning |
|---|---|
| `0` | Disable process address-space randomization. |
| `1` | Enable partial randomization of relevant memory regions. |
| `2` | Additionally randomize the heap; this is the common full-ASLR setting. |

The precise behavior depends on the kernel architecture and configuration. In particular, the default can depend on the kernel's `CONFIG_COMPAT_BRK` option.

To change the setting temporarily, use a privileged shell:

```bash
sudo sysctl -w kernel.randomize_va_space=2
```

Verify the result:

```bash
sysctl kernel.randomize_va_space
cat /proc/sys/kernel/randomize_va_space
```

For a persistent configuration, create a file such as `/etc/sysctl.d/60-aslr.conf` containing:

```ini
kernel.randomize_va_space = 2
```

Apply and verify it:

```bash
sudo sysctl --system
sysctl kernel.randomize_va_space
```

**Security warning:** Setting this parameter to `0` weakens exploit mitigations. Do not disable ASLR on a production system merely to make a demonstration work.

## 6. Key Takeaways

- The CIA triad represents confidentiality, integrity, and availability.
- CVE identifiers provide standardized references to publicly disclosed vulnerabilities.
- Keeping software updated is important, but security depends on more than package updates alone.
- Memory corruption includes multiple bug classes; buffer overflow is one of them.
- Stack exhaustion and stack-based buffer overflow are different problems.
- `malloc()`, `calloc()`, and `free()` provide memory-management functions, not automatic memory safety.
- Stack canaries, NX/DEP, PIE, and ASLR provide complementary protections.
- `kernel.randomize_va_space=2` enables the full-ASLR setting where supported; `0` disables it.
- Security mitigations reduce risk, but preventing the underlying defect is preferable to relying on exploit mitigations alone.

---

To understand memory corruption, buffer overflows, and ASLR, you should understand how a running program uses memory. Let's focus on the concepts you actually need for Linux security.

## A process's memory layout
When you run a program, Linux creates a process with its own virtual address space. That address space contains different regions used for different purposes.

### Simplified process memory layout

Conceptual diagram — not to scale; exact layout varies.

```
---------------------------------------------------------
|                         Stack                         |
|   Function calls, local variables, return addresses   |
---------------------------------------------------------
                           ↓↑                             
        May grow or expand toward other regions                                               
---------------------------------------------------------
|                 Memory-mapped regions                 |
|    Shared libraries, mapped files, other mappings     |
---------------------------------------------------------
---------------------------------------------------------
|                         Heap                          |
|              Dynamically allocated memory             |
---------------------------------------------------------
---------------------------------------------------------
|                     Data and BSS                      |
|              Global and static variables              |
---------------------------------------------------------
---------------------------------------------------------
|                   Text (code)                         |
|               Program instructions                    |
---------------------------------------------------------
Illustrative arrangement only. Modern processes may use different layouts and additional mappings.
```

The key regions to know are the stack, heap, data/BSS, and text (code). The mapped regions are also relevant to shared libraries and ASLR.

- ## Stack

    The stack is a memory region used to manage function calls and their temporary data.

    For example:

    ```c
    void example(void)
    {
        int age = 25;
        char name[100];
    }
    ```


    When `example()` runs, its local variables typically live in a stack frame on the stack.

    The stack is also used for information needed to manage **function calls**, such as **return addresses** and **saved register values**, depending on the architecture and compiler.

    **Important properties:**

    - Stack frames are typically created when functions are called and released when they return.
    - Each thread normally has its own stack.
    - Stack space is limited; excessive recursion or large stack allocations can exhaust it.
    - A stack-based buffer overflow occurs when a program writes beyond a buffer located on the stack.

    **Security connection:** A stack-based buffer overflow can overwrite nearby data in a stack frame and, in some circumstances, interfere with program control flow.

    > ## What is a function call?
    >
    > A function is a named block of code that performs a task. A function call means asking the program to execute that function.
    >
    > For example:
    >
    > ```c
    > #include <stdio.h>
    > 
    > void greet(void)
    > {
    >     printf("Hello!\n");
    > }
    > 
    > int main(void)
    > {
    >     greet();  // Function call
    >     return 0;
    > }
    > ```
    > 
    > When the program reaches `greet()`;, execution temporarily moves from `main()` to `greet()`. After `greet()` finishes, execution returns to the statement after the call in `main()`.
    >
    > The flow is:
    >
    > ```
    > ---------------------------------
    > | 1. `main()`                   |
    > | Reaches `greet();`            |
    > ---------------------------------
    >                ↓
    > ---------------------------------
    > | 2. `greet()`                  |
    > | Executes printf("Hello!\n");  |
    > ---------------------------------
    >                ↓
    > ---------------------------------
    > | 3. Back to `main()`           |
    > | Continues with `return 0;`    |
    > ---------------------------------
    > ```
    > 
    > ## What is a return address?
    >
    > The CPU needs to know where to continue executing after a function finishes. The address of that next instruction is called the return address.
    >
    > For example:
    > ```c
    > int main(void)
    > {
    >     greet();       // Call happens here
    >     printf("Done\n");  // Continue here after greet() returns
    > }
    > ```
    > 
    > Conceptually, the CPU must remember the location of the instruction that executes `printf("Done\n");` so that it can resume there after `greet()` finishes.
    >
    > On many common architectures and calling conventions, the return address is saved on the stack as part of the function-call process. On others, it may initially be held in a CPU register and saved elsewhere when necessary. The exact mechanism depends on the architecture and calling convention.
    >
    > ### How does this relate to stack frames?
    > When a function is called, it may need space for its local variables, saved registers, and other information. This information is commonly organized into a **stack frame**.
    >
    > For example:
    >
    > ```c
    > void calculate(void)
    > {
    >     int result = 10;
    > }
    > 
    > int main(void)
    > {
    >     calculate();
    >     return 0;
    > }
    > ```
    > 
    > While `calculate()` executes, its stack frame may contain the local variable `result` and other function-call information. When the function returns, its stack frame is no longer active, and execution resumes in `main()`.
    >
    > **One important distinction:** the compiler and calling convention determine exactly what goes into a stack frame. Not every local variable is necessarily stored there.
    >
    > ### Why do return addresses matter in security?
    > This connects directly to your LPIC-3 303 memory-corruption material.
    >
    > A stack-based buffer overflow occurs when a program writes beyond the bounds of a buffer allocated on the stack. Depending on the stack layout, the overflow might overwrite other local data or control-related information, potentially including a saved return address.
    >
    > If an attacker can corrupt the return address and bypass relevant protections, they may be able to redirect program execution. This is one historical route to code execution vulnerabilities.
    >
    > Modern protections include:
    > - **Stack canaries**: Detect certain stack overwrites before a function returns.
    > - **NX/DEP**: Prevent execution from memory regions that should contain data only.
    > - **ASLR**: Randomizes the locations of memory regions.
    > - **PIE**: Allows the executable's code location to be randomized.
    >
    > These protections make exploitation harder, but they do not fix the underlying out-of-bounds write.
    >
    > **The main idea to remember:** the stack is used not only for local variables, but also for managing function calls. That is why corrupting stack memory can sometimes affect where a program executes next.


    > ## Where Is User Input Stored in Memory During Program Execution: Stack or Heap?
    >
    > The memory location depends on how the program stores the user's input, not on the fact that the user entered it while the program was running.
    >
    > Let's look at three examples in C.
    >
    > - **User input stored in a local array — typically stack**
    >
    >   ```c
    >   #include <stdio.h>
    >   
    >   int main(void)
    >   {
    >       char name[100];
    >   
    >       scanf("%99s", name);
    >   
    >       return 0;
    >   }
    >   ```
    >   
    >   Here, `name` is a local array, typically stored on the stack. The user's input is written into that array, so the input is stored in stack memory.
    >
    > - **User input stored in dynamically allocated memory — heap**
    >
    >   ```c
    >   #include <stdio.h>
    >   #include <stdlib.h>
    >   
    >   int main(void)
    >   {
    >       char *name = malloc(100);
    >   
    >       if (name == NULL)
    >           return 1;
    >   
    >       scanf("%99s", name);
    >   
    >       free(name);
    >       return 0;
    >   }
    >   ```
    >   
    >   Here, `name` is a local pointer variable, typically on the stack, but the 100-byte memory block allocated by `malloc()` is on the heap. The user's input goes into that heap allocation.
    >
    > - **User input stored in a global array — data/BSS region**
    >   
    >   ```c
    >   #include <stdio.h>
    >   
    >   char name[100];
    >   
    >   int main(void)
    >   {
    >       scanf("%99s", name);
    >   
    >       return 0;
    >   }
    >   ```
    >   
    >   Here, `name` is a global array, so it has static storage duration and is typically placed in the BSS region.
    >
    > ### Summary
    >
    > | Where the program stores input | Typical memory location |
    > | --- | --- | 
    > | Local array, e.g. `char name[100]` inside a function | Stack |
    > | Memory allocated with `malloc()` | Heap |
    > | Global or static array | Data/BSS region |
    >
    > The key distinction is that user input is just data; the program determines where that data is stored. Input does not automatically go to the stack or heap simply because it was entered interactively.
     
    ---

- ## Heap

    The heap is a memory region used for dynamic memory allocation.

    Example in **`C`**:

    ```c
    #include <stdlib.h>

    int main(void)
    {
        int *number = malloc(sizeof(int));

        if (number == NULL)
            return 1;

        *number = 42;

        free(number);
        return 0;
    }
    ```

    Here:

    - number is a local pointer variable, typically stored on the stack.
    - `malloc()` allocates memory dynamically, typically from the heap.
    - `*number` accesses the allocated integer.
    - `free(number)` releases the allocation.

    Notice that the pointer and the memory it points to are two different things.

    > ### deeper Dive:
    > 
    > The key is that `number` and the memory it points to are two different things.
    >
    > - **Stack — local pointer variable**
    > 
    >   Conceptually, `number` stores a memory address.
    >
    >   ```c
    >   number
    >   0x1000
    >   Example address stored in the pointer
    >   ```
    > 
    >   The stored address points to the allocated memory.
    > 
    > - **Heap — dynamically allocated integer**
    > 
    >   ```c
    >   Example address: 0x1000
    >   42
    >   The actual integer value stored in the allocated memory
    >   ```
    > 
    > The addresses above are illustrative. Here's what happens:
    >
    > 1. `int *number` declares a pointer variable named number.
    > 2. `malloc(sizeof(int))` requests enough dynamically allocated memory to hold an integer.
    > 3. `malloc()` returns the address of that memory.
    > 4. The address is stored in `number`.
    > 5. `*number = 42` writes 42 into the allocated memory, not into the pointer variable itself.
    >
    > So, in a typical implementation:
    >
    > | Expression | What it represents | Typical location |
    > | --- | --- | --- |
    > | **`number`** | The pointer variable holding an address | Stack |
    > | **`malloc(sizeof(int))`** | The allocated memory | Heap |
    > | **`*number`** | The integer stored at the pointed-to address | Heap |
    >
    > **Important nuance:** **`C`** does not require local variables to be physically stored on the stack. A compiler may keep a variable in a CPU register or optimize it away. But for understanding the usual memory model, thinking of this local pointer as being on the stack is correct.
    >
    > Also, `free(number)` releases the allocated heap memory; it does not remove the local pointer variable itself. The local variable's lifetime ends when its scope ends.
    > 


    **Important properties:**

    - Memory can remain allocated after the function that requested it returns.
    - In `C`, the programmer must manage its lifetime.
    - Failing to release memory can cause a memory leak.
    - Accessing memory after releasing it can cause a use-after-free vulnerability.
    - Writing beyond a heap allocation can cause a heap-based buffer overflow.

    > **Memory leak**: Occurs when a program allocates memory but no longer needs it and fails to release it, causing memory usage to grow over time.
    >
    > **Memory spraying**: An exploitation technique in which an attacker fills large portions of memory with carefully crafted data to increase the chance that a vulnerability redirects execution to attacker-controlled content.

    ### Stack vs. heap
    | Feature | Stack | Heap |
    | --- | --- | --- |
    | Main purpose | Function calls and temporary data | Dynamic memory allocation |
    | Typical allocation | Automatically managed stack frames | Explicit allocation in `C` using `malloc()` or `calloc()` |
    | Lifetime | Usually associated with function calls | Until released or the process terminates |
    | Management | Mostly automatic | Programmer-managed in `C` |
    | Common problems | Stack exhaustion, stack-based buffer overflow | Memory leaks, use-after-free, heap-based buffer overflow |
    
    > Do not memorize that stack memory is always faster or that heap memory is always slower. Performance depends on the allocation method, workload, and implementation.

    ---

- ## Text, data, and BSS

    These regions are useful to recognize, particularly when learning about memory layout and ASLR.

    - **Text (code):** Contains the program's machine instructions. It is commonly mapped as readable and executable, but not writable.
    - **Data segment:** Holds initialized global and static variables.
    - **BSS:** Holds uninitialized or zero-initialized global and static variables. These variables start with zero values.

    Example:

    ```c
    int global_initialized = 10;
    int global_zero;

    int main(void)
    {
        static int counter = 5;
        return 0;
    }
    ```

    The global variables and `counter` have static storage duration. Their storage lasts for the lifetime of the process; they are not ordinary local stack variables.

## Virtual memory vs. physical memory

**Virtual memory** is the address space that a process uses. Physical memory is the actual RAM installed in the machine.
The operating system and the CPU's memory-management hardware translate virtual addresses to physical memory locations. Some virtual memory may also be backed by files or swap rather than currently resident in RAM.

Each process typically has its own virtual address space, so two processes can use the same virtual address while referring to different physical memory.

**Security connection:** ASLR randomizes the locations of certain regions in a process's virtual address space. This makes it harder for an attacker to predict where particular data or code will be located.

###  How these concepts connect to buffer overflows

Suppose a program declares:

```c
char buffer[100];
```

This reserves space for 100 bytes. Where the buffer is located depends on how it is declared and used:

- A local array like this is typically on the stack.
- A buffer allocated using `malloc()` is typically on the heap.
- A global or static array may reside in the data or BSS region.

If the program writes beyond the buffer's 100-byte boundary, it causes an out-of-bounds write. The consequences depend on where the buffer is located, what memory is adjacent to it, and which protections are enabled.

This is why the terms stack-based buffer overflow and heap-based buffer overflow exist.

### Suumery

- **Stack** → Function calls and local data. Know stack frames, stack exhaustion, and stack-based buffer overflows.
- **Heap** → Dynamic allocation. Know `malloc()`, `calloc()`, `free()`, memory leaks, use-after-free, and heap-based buffer overflows.
- **Virtual address space** → The memory addresses a process uses. Know that ASLR randomizes the locations of relevant regions.
- **Memory protections** → Know ASLR, stack canaries, NX/DEP, and PIE — and understand that these reduce exploitability but do not fix the underlying bug.