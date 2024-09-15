---
title: "(Linux) Convert pem to ppk"
date: 2018-02-14T00:28:21+04:00
draft: false
toc: false
images:
tags:
  - linux
  - pem
  - ppk
  - puttygen
---
# Convert PEM to PPK & PPK to PEM

If you're using AWS EC2, you may need to convert PPK to PEM (or vice versa) when trying to SSH from different operating systems. Below are instructions on how to perform these conversions using the `puttygen` tool.

## Convert PPK to PEM

1. **Install `puttygen`**: 
   - On **Linux**: You can install `putty-tools` using your package manager. For example, on Debian-based systems, run:
     ```sh
     sudo apt-get install putty-tools
     ```
   - On **Windows**: Download and install PuTTY from the [official website](https://www.putty.org/).

2. **Convert the file**:
   - Open your terminal or command prompt.
   - Run the following command to convert the PPK file to PEM format:
     ```sh
     puttygen your-key.ppk -O private-openssh -o your-key.pem
     ```
   Replace `your-key.ppk` with the name of your PPK file, and `your-key.pem` with the desired name for your PEM file.

## Convert PEM to PPK

1. **Install `puttygen`**: Follow the same installation steps as above.

2. **Convert the file**:
   - Open your terminal or command prompt.
   - Run the following command to convert the PEM file to PPK format:
     ```sh
     puttygen your-key.pem -O private -o your-key.ppk
     ```
   Replace `your-key.pem` with the name of your PEM file, and `your-key.ppk` with the desired name for your PPK file.

## Summary

- **PEM to PPK**: Use `puttygen your-key.pem -O private -o your-key.ppk`
- **PPK to PEM**: Use `puttygen your-key.ppk -O private-openssh -o your-key.pem`

These conversions are particularly useful for managing SSH keys across different systems and applications. Ensure that you handle and store your private keys securely.
