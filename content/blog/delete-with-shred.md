+++
title = 'Permanently erase sensitive files with shred'
date = 2024-03-03T21:55:52+02:00
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
`shred`. Instead, I will attempt to show how to counter (on a BTRFS filesystem, since I use Fedora) a pitfall that the 
article calls a crucial assumption. I will also write a simple script to use the program on directories.

This crucial assumption is that the file system overwrites data in place. This means that when a file is modified, the
changes are written directly to the same location of the data. This might be favourable on systems that have high write
workloads. On the contrary, if the user needs data integrity or recovery, he might choose to use a copy-on-write
(CoW) file system like BTRFS. When data is modified on a CoW file system, it is first copied to a new location, and the 
changes are then made to that copy. `shred` does not warn about this pitfall, and it's the user's responsibility to 
ensure that the files are overwritten as expected.

I found two methods to avoid CoW on files I want to get rid of for good, out of which only the second one will work for
files

## Changing file attributes with `chattr`

The first is to change the file's attributes using `chattr`. File attributes are metadata that tells the
operating system how to work with the file. For example, `chattr +i something.txt` makes the file read-only, while `chattr +C something.txt`
will signal that the file is not subject to CoW. This is the attribute I need - so I thought. As I mentioned, the 
file system
of my root partition is BTRFS. Here is what the man page of `chattr` says about my case:

> Note: For btrfs, the 'C' flag should be set on new or empty files.
> If it is set on a file which already has data blocks, it is undefined when the blocks assigned to the file will
> be fully stable.

This means when creating an empty file, I should be able to set the `C` attribute.

```
$ touch something.txt
$ lsattr something.txt
---------------------- something.txt
$ chattr +C something.txt
---------------C------ something.txt
```

Works as expected. Now let's see what happens if I try to set `C` on a file which already has some content:

```
$ echo Hello there! > something2.txt
$ chattr +C something2.txt
chattr: Invalid argument while setting flags on something2.txt
```

Also works as expected, which means I can't use `chattr` on existing files to avoid CoW when shredding. My guess is 
that this is because BTRFS cannot retroactively remove possible existing shadow copies. But if you know I'm wrong, 
please let me know.

## Mounting with `-o nodatacow`

Mounting with `-o nodatacow` will disable CoW for all subvolumes.

```
sudo mkdir /mnt/btrfs-drive && sudo mount -o nodatacow /dev/sda1 /mnt/btrfs-drive/
```



## Scripting with `shred`


a script that removes all regular files and directories inside the directories


However, I quickly learnt that this script will result in `Warning: Program '/bin/bash' crashed.` when invoked on deep directories.
The cause is probably stack overflow, because of the recursion. A quick look at the systemd journal verified this.


`shred` accepts multiple files as arguments, but not directories. 

`shred` works with symbolic links too, 


When `find` is invoked on a file with `-type f`

```bash
#!/bin/bash

for path in "$@"; do
    if [ -f "$path" ] || [ -d "$path" ]; then
        if [ -h "$path" ]; then
            link="$path"
            path=$(readlink "$link")
            unlink "$link"
        fi
        find "$path" -type f -exec shred -uzvf {} +
    else
        echo "Not regular file or directory: $path"
    fi
done
```

Then, you can add this function to your `.bashrc`, or make it a standalone script. I placed it in `.local/bin`.



[^1]: https://en.wikipedia.org/wiki/Gutmann_method

[^2]: https://git.savannah.gnu.org/cgit/coreutils.git/tree/src/shred.c

[^3]: https://git.busybox.net/busybox/tree/coreutils/shred.c

[^4]: https://www.gnu.org/software/coreutils/manual/html_node/shred-invocation.html