---
title: SUID Exploitation - How Privileged Programs Can Be Weaponized
metaDescription: 
date: 2026-01-31T02:00:00.000Z
summary: Talks about how SUID bit set programs can be exploited.
redirect: https://x.com/bbetterengineer/status/2013506472173543649
tags:
  - cybersecurity
  - infosec
  - linux
---

I don't know how to begin as this is my first long writeup. But what I'm going to tell you will blow your mind.

**THE FOUNDATION: UNDERSTANDING PERMISSIONS**

Sometimes, whether intentionally or unintentionally, we leave our systems insecure by giving programs more privileges than they actually require.

Think about it: we can execute binaries like cat, find, tail, and head. But if you check the owner and group info of these programs, you'll find root. So how are we able to execute them?

The answer lies in permissions. When you check the permissions, your user (the output of whoami) has the permissions to execute these programs.

**Breaking Down Permission Triplets**

The permission string -rwxrwxrwx breaks down like this:
- **First triplet (rwx):** Owner permissions
- **Second triplet (rwx):** Group permissions
- **Third triplet (rwx):** All others permissions

Each permission has a numeric value: read (4), write (2), execute (1).

**How File Access Actually Works**

When you invoke cat file.txt, it displays the content of the file. But not every file can be displayed with cat. You can only see the content if you have permissions to read the file, regardless of who the owner is.

If you don't have read permissions on file.txt, you can't use cat even if you have execute permissions on the cat binary itself. You'll end up with a "permission denied" error.

You need one of these conditions to be true:
- You (your username) are the owner and the owner has read PERMISSIONS.
- You're in the group and the group has read permissions
- If neither owner nor group, then "all others" (third triplet) should have read permissions

**The Real Part: SUIDBit Exploitation**

Here's where things get interesting.

When we execute any program, it has two kinds of owner information associated with it:
- **Real user:** The person who is executing the program
- **Effective user:** The one which is set and visible when you run ls -lah

**When SUID Changes Everything**

Some programs (pay close attention here) have the Set User ID (SUID) bit set. Here's what this means and how to identify it:
```bash
ls -lah cat
-rws r-x r-- root root
```

Notice in the first triplet there's an **s instead of x**. This means when I (as a non-root user) execute this program, the real user becomes root, not me.

But wait-I just said the real user is the one who is executing the program. That's true, but only when **there is x instead of s**.

When the SUID bit is set:
- The **s** indicates that the program will run with the permissions of the effective user (root).
- It does NOT run with the permissions of whoever is executing it.
- The privilege level of the program just increased because it's running with root privileges.

**The Dangerous Difference**

**With SUID bit (s):**
```bash
-rws r-x r-- root root
```

When I (non-root) run this program, it executes with root privileges.

**Without SUID bit (x):**
```bash
-rwx r-x --- root root
```

When I (non-root) run this program, it runs with my limited privileges.

**This bit change from x to s is extremely dangerous.** Generally, you won't find many binaries on your system with this s bit set. If you do find any suspicious ones, investigate and remove the SUID bit immediately.

#### Real-World Exploitation Examples

Let me show you how SUID-enabled programs can be exploited to access sensitive system information.

**Case 1: Genisoimage**
- Normally, this program drops privileges by running:
```bash
setuid(getuid());
```

- However, certain parameters can be passed that prevent it from dropping privileges immediately.
- **The vulnerability:** If you supply the -sort option, it will read file content before dropping privileges.
- **Exploitation:**
```bash
genisoimage -sort /etc/sensitive_file
```
- **Pattern to remember:** Find options that read data without dropping privileges first.

**Case 2: Find**
- This program doesn't drop privileges at all. If we can find a way to read protected files, we're in.
- The find command locates files by name and can chain with other programs to take action.
- **Exploitation:**
```bash
find /etc -maxdepth 1 -type f -name 'shadow' -print -exec cat {} \;
```
- **What's happening:**
    - Finds the file named shadow in /etc
    - Executes cat on this file
    - Doesn't recurse (maxdepth = 1)
    - All happens in escalated privilege mode because find has SUID set

- You might think cat doesn't have enough privileges, but that doesn't matter. When you invoked find, it got root privileges, and cat inherits those privileges as a child process.
- **Pattern to remember:** Check if the program can execute another program.

**Case 3: Wget**
- This one is trickier. You might think wget is just for downloading internet resources. But consider authentication-some resources require username and password.
- **The vulnerability:** The --use-askpass option expects you to provide a binary or executable from where it will get the password.
- **Exploitation:**
```bash
mktemp
chmod +x <temp_file>
echo -e '#!/bin/sh -p\n/bin/sh -p 1>&0' > tmp_file
wget --use-askpass=temp_file 0
```
- **What's happening:**
    - Creates a temporary executable file
    - Tricks wget into executing a user-supplied program as an "askpass helper"
    - If wget runs with elevated privileges, the helper inherits them
    - The helper spawns a privileged shell

- **Pattern to remember:** Find options that accept helper programs or external executables.

**Case 4: SSH-keygen**
- This requires providing custom compiled code in .so format.
- **Exploitation:**
```bash
ssh-keygen -D <.so file>
```
- The -D option expects a PKCS11 provider. You can write a C program to read protected files, compile it as a shared object, and pass it to ssh-keygen.
- **Pattern to remember:** Look for options expecting providers or helper libraries.

**Other Vulnerable Binaries**

Some programs are more straightforward. If the SUID bit is set on binaries like head, tail, awk, or sed, you can directly recover sensitive file content.

However, some programs require diving into their documentation, reading all available options, and checking if any can be exploited.

So here is a python code to find the binaries which have SUID bit set on them.

```python
#!/usr/bin/env python3

import os
import stat

SEARCH_DIRS = [
    "/bin",
    "/sbin",
    "/usr/bin",
    "/usr/sbin",
    "/usr/local/bin",
    "/usr/local/sbin",
]


def find_suid_binaries():
    results = []

    for base in SEARCH_DIRS:
        if not os.path.isdir(base):
            continue

        for root, dirs, files in os.walk(base):
            for name in files:
                path = os.path.join(root, name)
                try:
                    st = os.stat(path)
                except (PermissionError, FileNotFoundError):
                    continue

                if stat.S_ISREG(st.st_mode) and (st.st_mode & stat.S_ISUID):
                    results.append(path)

    return results


if __name__ == "__main__":
    suid_bins = find_suid_binaries()

    for path in suid_bins:
        print(path)

    print(f"\nTotal SUID binaries found: {len(suid_bins)}")
```

#### The Key Takeaway
The key to SUID exploitation is understanding:
- How privilege escalation works through the SUID bit
- Which programs have SUID set (audit your system regularly)
- Program options that allow file reading or command execution
- Parent-child process privilege inheritance

**Defensive measures:**
- Audit binaries with SUID bits regularly
- Remove unnecessary SUID bits
- Monitor for suspicious SUID changes
- Understand what each SUID program does and why it needs elevated privileges

This isn't just theoretical-these vulnerabilities exist in real systems. Follow me on X to get more such content on low level OS primitives, concurrency and networking.
