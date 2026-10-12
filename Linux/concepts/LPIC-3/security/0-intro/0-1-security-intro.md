# Security Fundamentals

> **Scope:** Supplementary introductory material. This section provides foundational security knowledge; it is not a standalone LPIC-3 303 exam objective.

This section introduces foundational information-security concepts, common software vulnerabilities, memory-corruption bugs, and Linux security updates.

## What Is Information Security?

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

Information security (InfoSec) protects information and the systems that store, process, and transmit it. A common foundational model is the **CIA triad**, consisting of three objectives:

| Objective | Meaning | Example |
|---|---|---|
| **Confidentiality** | Preventing unauthorized disclosure of information. | Restricting access to private keys and sensitive files. |
| **Integrity** | Preventing or detecting unauthorized or unintended modification or destruction of information. | Verifying package signatures and file checksums. |
| **Availability** | Ensuring authorized users can access systems and information when required. | Redundant services, backups, and protection against denial-of-service attacks. |

These objectives can conflict. For example, restricting access too aggressively may protect confidentiality but prevent legitimate users from accessing a service.

Security controls should therefore be selected according to the system's risks and requirements. The CIA triad is a foundational model, not an exhaustive list of every security objective.

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
