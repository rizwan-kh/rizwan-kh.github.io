---
title: "(Linux) Debug Shell Scripts"
date: 2018-05-01T12:30:00+04:00
draft: false
toc: false
images:
tags:
  - linux
  - debug
  - shell
  - scripts
---



# Debug Shell Scripts

In my experience, people when they want to debug a shell script, they usually use the `set +xv` and all this does is print each operation on the console output.

I found a better way to debug the shell script, which is consistent with going over each line on the press of the enter key

Simply add the below line in front of the place from where you want to start the debug

```sh
trap '(read -p "[$BASH_SOURCE:$LINENO] $BASH_COMMAND")' DEBUG
```

This command sets a `trap` for the `DEBUG` signal, which pauses the script and prompts you with the current source file, line number, and command being executed. You can press Enter to proceed to the next line.

## Example Usage

1. Insert the `trap` command at the desired location in your script:

```sh
#!/bin/bash

echo "Starting script..."

trap '(read -p "[$BASH_SOURCE:$LINENO] $BASH_COMMAND")' DEBUG

# Your script code here
echo "This is a test line."
var=2
echo $((var+2))
```

2. Run your script as usual. The script will pause at each line where the `trap` command is active, allowing you to inspect the execution flow interactively.

## Summary
- Debugging Method: Use `trap '(read -p "[$BASH_SOURCE:$LINENO] $BASH_COMMAND")' DEBUG` to pause and inspect script execution line-by-line.

This approach offers a more interactive way to debug shell scripts compared to traditional methods, giving you better control over script execution.