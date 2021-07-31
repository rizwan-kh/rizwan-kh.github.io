---
title: "tmux"
date: 2019-06-13T21:53:35+04:00
draft: false
toc: false
images:
tags:
  - tmux
  - terminal
---
![tmux](/tmux-logo.png)
# tmux - Terminal Multiplexer
tmux is a terminal multiplexer. It helps switch between multiple programs in one terminal, detach them(they keep running in background) and reattach when needed. I started using tmux as someone recommended me to start using it as it helps with CKA & CKAD (which I  have still not attempted) and then later on I was heavily using it for my website(with Hugo). I had previously used mPutty, but mPutty uses mutliple logged in sessions to display on screen, whereas tmux simply multiply the existing sessions on screen.

### Installation
Major distribution of linux provides tmux packages via standard pre-built packages of tmux.

|Platform   | Install Command |
| ----------- | ----------------- |
|Debian or Ubuntu | `apt install tmux` |
|RHEL or CentOS | `yum install tmux`|
|macOS (using Homebrew) | `brew install tmux` |

### Basic commands and usage

| Command | Description|
| :------- | :-----------|
| tmux | start a new session|
| tmux new -s my-kube-session | start a new session with name |
| tmux a | attach |
| tmux a  -t my-kube-session | attach to a named session |
| tmux ls | list all tmux sessions |
| tmux kill-session -t my-kube-session | kill the session named my-kube-session |

After a session is created, inside to perform any action, we need to hit `ctrl+b` followed by any below command/keystroke

|Keystroke|Description|
|:---------|:-----------|
|ctrl+b -> c|create new shell|
|ctrl+b -> n|next shell|
|ctrl+b -> p|previous shell|
|ctrl+b -> d|detach session|
|ctrl+b -> %|vertical split|
|ctrl+b -> "|horizontal split|
|ctrl+b -> [arrow keys]|to switch between the vertical/horizontal panes|
|ctrl+b -> z|zoom in and zoom out between split panes|


That's all more or less the keystroke you need to know; It comes of naturally when you start using it after a few days but until then - you can refer these as needed.