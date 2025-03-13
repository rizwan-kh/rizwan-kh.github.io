---
title: "Delve - Go Debugger"
date: 2025-03-13T10:40:00+04:00
draft: false
toc: false
images:
tags:
  - go
  - debugger
  - delve
  - dlv
---
## Introduction

[Delve](https://github.com/go-delve/delve) is a powerful debugger for Go(Golang). 

## Installation

To install latest version of Delve, use the below command

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

Make sure that `$GOPATH/bin` or your Go binary path is in your `PATH` environment variable.

## Quick Usage

You can use delve in many ways, the one I prefer is as below

- Start a Program with Delve

    You can start a Go program under Delve by running:

    ```bash
    dlv debug <program.go>
    ```

    This will compile and run the program in debug mode, allowing you to set breakpoints and interact with it.

- Attach to a Running Process

    If your Go program is already running, you can attach Delve to it by using the process ID (PID):

    ```bash
    dlv attach <pid>
    ```

### Basic Commands

Once delve is running, you can use the below commands to move around

- Set breakpoints
    
    ```bash
    break <function_name>
    ```

    Or for specific lines

    ```bash
    break <filename>:<line_number>
    ```

- Run program
    ``` bash
    continue
    ```

- Step through the code

    - Step into function: `step`
    - Step over function: `next`
    - Step out of current function: `stepout`

- Inspect variables
    - Print variable: `print <variable_name>`
    - Display the value of a variable at a breakpoint.

- List breakpoints
    
    ```bash
    breakpoints
    ```
- Exit the debug mode
    
    ```bash
    quit
    ```