+++
title = 'Permanently erase sensitive files with shred'
date = 2024-05-23T08:00:00+00:00
draft = false
tags = [
    "linux",
    "bash"
]
+++

If you're a little paranoid, like me, you might've wondered what really happens to 
the files you remove on a Linux operating system. Isn't it suspicious that when `rm` is invoked, it typically
returns code 0 in an instant, even when invoked on large files? That is because what really happens is that the file
is unlinked from the file system, but its data will still exist on the disk until the space is allocated to something else.
This obviously has two major advantages, performance and recovery. However, when dealing with files with sensitive content, 
I like to use `shred` for deletion. Not because the CIA is after me, or because I would leave my laptop unattended and unlocked, but simply because it 
makes me feel better.

The description of `shred` in its manpage is

> Overwrite  the  specified  FILE(s) repeatedly, in order to make it harder for even
> very expensive hardware probing to recover the data.

It uses the Gutmann[^1] method to repeatedly erase the contents of the file, and it is part of GNU coreutils[^2] and BusyBox[^3] as well. 
There's an excellent writing on `shred` on gnu.org[^4]. My favourite part is when the writer casually states

> The best way to remove something irretrievably is to destroy the media it’s on with acid, melt it down, or the like.

As if the average user would have a vat of acid at hand.

Because this article was written by someone much smarter than I, I will not try to go in-depth about the workings of 
`shred`. Instead, I will only attempt to show a simple script to use the program on directories.

{{< rawhtml >}}
<fieldset>
<legend><b>Warning</b></legend>

As the article on gnu.org mentions, <code>shred</code> is not a silver bullet on file systems that do not overwrite 
data in place. For example, BTRFS uses copy-on-write (CoW), which means that instead of directly writing to the exact 
location of the data,
it is first copied to a new location, and the changes are then made to that copy. If you use a file system with
CoW mechanism, change the file's attributes with <code>chattr +C file</code> before shredding it. Alternatively, 
if you are using BTRFS, you can mount the file system with <code>-o nodatacow</code> to disable CoW for all subvolumes.
Also, make sure you don't have any snapshots containing the file you want to get rid of.
</fieldset>
{{< /rawhtml >}}

A simple, practical use of `shred` would be

```
$ echo Destroy me! > something.txt
$ shred -uzfv -n 5 something.txt 
```

{{< rawhtml >}}
<details>
<summary>Click to toggle output</summary>
<code style="white-space: pre-wrap">
shred: something.txt: pass 1/6 (random)...
shred: something.txt: pass 2/6 (000000)...
shred: something.txt: pass 3/6 (random)...
shred: something.txt: pass 4/6 (ffffff)...
shred: something.txt: pass 5/6 (random)...
shred: something.txt: pass 6/6 (000000)...
shred: something.txt: removing
shred: something.txt: renamed to 0000000000000
shred: 0000000000000: renamed to 000000000000
shred: 000000000000: renamed to 00000000000
shred: 00000000000: renamed to 0000000000
shred: 0000000000: renamed to 000000000
shred: 000000000: renamed to 00000000
shred: 00000000: renamed to 0000000
shred: 0000000: renamed to 000000
shred: 000000: renamed to 00000
shred: 00000: renamed to 0000
shred: 0000: renamed to 000
shred: 000: renamed to 00
shred: 00: renamed to 0
shred: something.txt: removed
</code>
</details>
{{< /rawhtml >}}

- `-u` removes the file after overwriting
- `-z` adds a final overwrite with zeroes to hide shredding
- `-f` tries to change permissions to allow writing if necessary
- `-v` is verbose mode for more outputs

The default number of overwrites is 3, plus 1 if we add zeroes with `-z` to hide shredding.
I run Fedora, which uses BTRFS, but I do not have any snapshots:

```
$ sudo btrfs subvolume list /
ID 256 gen 81984 top level 5 path home
ID 257 gen 81984 top level 5 path root
ID 258 gen 81265 top level 257 path var/lib/machines
```

so I am reassured that the file is gone for good.

It would be convenient to have the ability to run shred on directories. I could write a recursive function that would
run `shred` on files, but call itself on directories. As much as I like recursion, I know this approach would result in
a stack overflow on deep directories. Instead, I will utilise `find`'s `-exec` option.

```bash
#!/usr/bin/env bash

for path in "$@"; do # Let's accept multiple files/directories
    if [ -f "$path" ] || [ -d "$path" ]; then
        # If the path is a symlink, unlink it and shred the target as well
        if [ -h "$path" ]; then
            link="$path"
            path=$(readlink "$link")
            unlink "$link"
        fi
        # When invoked on the file itself, find will return the path anyway
        find "$path" -type f -exec shred -uzvf {} +
    else
        # Shredding devices is out of scope for this little script
        echo "Not regular file or directory: $path"
    fi
done
```

Now, I could make this a function in my `.bashrc`, or save it as a standalone script. I chose the latter, and placed
it in `.local/bin`.

## To-do

---
- [ ] elaborate on the pitfalls of CoW/journaling when using `shred`
- [ ] elaborate on Gutman method
- [ ] extend script to check for file system type
- [ ] extend script to shred devices

## References

[^1]: [Gutman method - wikipedia.org](https://en.wikipedia.org/wiki/Gutmann_method)

[^2]: [shred.c - git.savannah.gnu.org](https://git.savannah.gnu.org/cgit/coreutils.git/tree/src/shred.c)

[^3]: [shred.c - git.busybox.net](https://git.busybox.net/busybox/tree/coreutils/shred.c)

[^4]: [shred invocation - gnu.org](https://www.gnu.org/software/coreutils/manual/html_node/shred-invocation.html)