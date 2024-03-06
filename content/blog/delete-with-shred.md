+++
title = 'Permanently delete files and directories with shred'
date = 2024-03-03T21:55:52+02:00
draft = false
tags = [
    "outdoors"
]
+++

If you're a little paranoid, like me, you might've wondered what really happens to the files you deleted. Isn't it suspicious that even if you delete large directories with `rm -r`, the command returns 0 almost immediately?

When dealing with files with sensitive contents, I typically use `shred` for deletion. This program is part of GNU coreutils and BusyBox as well.

`shred` only works on files.

a script that removes all regular files and directories inside the directories


```bash
function destroy {
    for path in "$@"; do
        if [ -f "$path" ]; then
            shred -uzv "$path"
        elif [ -d "$path" ]; then
            destroy "$path" && rmdir -v "$path"
        elif [ -h "$path" ]; then
            unlink "$path"
        else
            echo "No such file or directory, or can't be removed: $path"
        fi
    done
}
```

However, I quickly learnt that this script will result in `Warning: Program '/bin/bash' crashed.` when invoked on deep directories. The cause is probably stack overflow, because of the recursion. A quick look at the systemd journal verified this.

{{< rawhtml >}}
  <details>
    <summary>
      Click to see systemd logs
    </summary>
    <p style="white-space: pre-line;">
        Stack trace of thread 33230:
        #0 0x0000556c9d0e5b10 param_expand.lto_priv.0 (bash + 0x7fb10)
        #1 0x0000556c9d0e792d expand_word_internal.lto_priv.0 (bash + 0x8192d)
        #2 0x0000556c9d0e7b90 expand_word_internal.lto_priv.0 (bash + 0x81b90)
        #3 0x0000556c9d0e9a71 expand_word_list_internal.lto_priv.0 (bash + 0x83a71)
        #4 0x0000556c9d0bbe14 execute_simple_command.lto_priv.0 (bash + 0x55e14)
        #5 0x0000556c9d0b0c04 execute_command_internal (bash + 0x4ac04)
        #6 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #7 0x0000556c9d0b008e execute_command_internal (bash + 0x4a08e)
        #8 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #9 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #10 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #11 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #12 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #13 0x0000556c9d0b1d0f execute_command_internal (bash + 0x4bd0f)
        #14 0x0000556c9d0b01bc execute_command_internal (bash + 0x4a1bc)
        #15 0x0000556c9d0b7816 execute_function (bash + 0x51816)
        #16 0x0000556c9d0bd144 execute_simple_command.lto_priv.0 (bash + 0x57144)
        #17 0x0000556c9d0b0c04 execute_command_internal (bash + 0x4ac04)
        #18 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #19 0x0000556c9d0b008e execute_command_internal (bash + 0x4a08e)
        #20 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #21 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #22 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #23 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #24 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #25 0x0000556c9d0b1d0f execute_command_internal (bash + 0x4bd0f)
        #26 0x0000556c9d0b01bc execute_command_internal (bash + 0x4a1bc)
        #27 0x0000556c9d0b7816 execute_function (bash + 0x51816)
        #28 0x0000556c9d0bd144 execute_simple_command.lto_priv.0 (bash + 0x57144)
        #29 0x0000556c9d0b0c04 execute_command_internal (bash + 0x4ac04)
        #30 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #31 0x0000556c9d0b008e execute_command_internal (bash + 0x4a08e)
        #32 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #33 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #34 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #35 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #36 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #37 0x0000556c9d0b1d0f execute_command_internal (bash + 0x4bd0f)
        #38 0x0000556c9d0b01bc execute_command_internal (bash + 0x4a1bc)
        #39 0x0000556c9d0b7816 execute_function (bash + 0x51816)
        #40 0x0000556c9d0bd144 execute_simple_command.lto_priv.0 (bash + 0x57144)
        #41 0x0000556c9d0b0c04 execute_command_internal (bash + 0x4ac04)
        #42 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #43 0x0000556c9d0b008e execute_command_internal (bash + 0x4a08e)
        #44 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #45 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #46 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #47 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #48 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #49 0x0000556c9d0b1d0f execute_command_internal (bash + 0x4bd0f)
        #50 0x0000556c9d0b01bc execute_command_internal (bash + 0x4a1bc)
        #51 0x0000556c9d0b7816 execute_function (bash + 0x51816)
        #52 0x0000556c9d0bd144 execute_simple_command.lto_priv.0 (bash + 0x57144)
        #53 0x0000556c9d0b0c04 execute_command_internal (bash + 0x4ac04)
        #54 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #55 0x0000556c9d0b008e execute_command_internal (bash + 0x4a08e)
        #56 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #57 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #58 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #59 0x0000556c9d0b0d69 execute_command_internal (bash + 0x4ad69)
        #60 0x0000556c9d0b2ece execute_command (bash + 0x4cece)
        #61 0x0000556c9d0b1d0f execute_command_internal (bash + 0x4bd0f)
        #62 0x0000556c9d0b01bc execute_command_internal (bash + 0x4a1bc)
        #63 0x0000556c9d0b7816 execute_function (bash + 0x51816)
        ELF object binary architecture: AMD x86-64 
    </p>
  </details>
{{< /rawhtml >}}





