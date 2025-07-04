---
title: "Solving the npm EACCES Permission Error on macOS: A Comprehensive Guide"
date: 2025-07-04
draft: false
summary: ""
tags: ["Node.js","npm","MacOS","Linux"]
---


Using Node.js and npm on macOS is a daily routine for web developers. However, a common roadblock nearly everyone hits is the `EACCES` permission error, especially when trying to install a global package using the `-g` flag.

Recently, I encountered this classic issue while trying to install the Google Gemini CLI. The error log looked like this:

```bash
shangshui@MacBook-Pro ~ % npm install -g @google/gemini-cli
npm error code EACCES
npm error syscall mkdir
npm error path /usr/local/lib/node_modules/@google
npm error errno -13
npm error Error: EACCES: permission denied, mkdir '/usr/local/lib/node_modules/@google'
...
npm error The operation was rejected by your operating system.
npm error It is likely you do not have the permissions to access this file as the current user
```

This blog post will dive into the root cause of this problem and provide three solutions, ranging from a quick fix to the industry best practice.

## Analyzing the Root Cause

The core of this error is straightforward: **Insufficient Permissions**. Let's break down the key pieces of information:

*   **`EACCES`**: An abbreviation for "Access Denied."
*   **`mkdir '/usr/local/lib/node_modules/@google'`**: npm attempted to create a new directory named `@google` inside the `/usr/local/lib/node_modules/` path, but it failed.
*   **The Reason**: On macOS, directories like `/usr/local/` are protected system-level locations. By default, a standard user (like `shangshui`) does not have the permission to write or create files in these directories. When you use `npm install -g`, npm tries to install the package into this protected area, and the operating system rightly denies the request.

## Solutions

There are three different ways to resolve this issue. I will present them in order from "not recommended" to "highly recommended."

### Method 1: The Quick (but Not Recommended) `sudo` Fix

This is the most direct approach, using the `sudo` command to grant npm temporary administrative privileges to perform the installation.

```bash
sudo npm install -g @google/gemini-cli
```

You will be prompted to enter your Mac's login password.

*   **Pros**:
    *   Simple and fast, it solves the immediate problem.
*   **Cons**:
    *   **Security Risk**: You are running a script downloaded from the internet with the highest level of system privileges. If the package contains malicious code, it could potentially harm your system.
    *   **Permission Mess**: Packages installed with `sudo` are owned by the `root` user. In the future, you might need `sudo` again to update or uninstall them, leading to confusing permission management.

### Method 2: Fixing npm's Directory Permissions (Recommended)

This method provides a permanent fix by changing the ownership of npm's global directory to your current user. After this, you will never need `sudo` for global installs again.

1.  **Find npm's default directory prefix**.
    ```bash
    npm config get prefix
    ```
    On most macOS systems, this will return `/usr/local`.

2.  **Change the ownership of the directory**. Use the `chown` command to recursively (`-R`) assign ownership of the relevant directories to your current user (`$(whoami)` is a command substitution that inserts your username).

    ```bash
    sudo chown -R $(whoami) $(npm config get prefix)/{lib/node_modules,bin,share}
    ```

    This command changes the ownership of three key subdirectories, allowing you to install global packages as a normal user.

*   **Pros**:
    *   A one-time fix that is secure and convenient.
    *   Follows one of the recommendations from npm's official documentation.
*   **Cons**:
    *   The command can seem a bit complex if you're not familiar with the command line.

### Method 3: The Best Practice - Using NVM (Highly Recommended)

**NVM (Node Version Manager)** is a tool for managing Node.js versions. Not only does it allow you to easily switch between different Node versions, but it also solves the permission problem at its root.

NVM works by installing Node.js and all global npm packages within your **user's home directory** (e.g., `~/.nvm/...`). Since you have full ownership of this directory, you will **never encounter an EACCES error**.

1.  **Install NVM**. Run the official installation script.
    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    ```

2.  **Configure your environment**. Close and reopen your terminal, or run `source ~/.zshrc` (or `~/.bash_profile`) to activate NVM.

3.  **Install Node.js with NVM**.
    ```bash
    nvm install --lts
    ```
    This will install the latest Long-Term Support (LTS) version and set it as your default.

4.  **Verify the installation**. Check the path to your `npm` executable:
    ```bash
    which npm
    # The output should be something like: /Users/shangshui/.nvm/versions/node/v20.12.2/bin/npm
    ```
    You'll notice it's no longer in `/usr/local/bin/`.

5.  **Install your package again**. Now, you can install any global package without any issues.
    ```bash
    npm install -g @google/gemini-cli
    ```

*   **Pros**:
    *   **Completely eliminates permission issues**.
    *   Allows for flexible management and switching of Node.js versions, which is crucial for development.
    *   Keeps your system directories clean and unmodified.
    *   It is the undisputed **best practice** in the Node.js community.

## Summary and Recommendation

| Method | Pros | Cons | Recommendation |
| :--- | :--- | :--- | :--- |
| **`sudo`** | Quick | Insecure, can mess up permissions | ★☆☆☆☆ |
| **Change Permissions** | Permanent fix, secure | Requires understanding `chown` | ★★★★☆ |
| **NVM** | Solves permission issues, version management, best practice | Requires extra setup | ★★★★★ |

While `sudo` can get you out of a jam, for long-term security and convenience, **using NVM is the highly recommended approach for all Node.js developers on macOS**. It sets you on the right path from the start, helping you avoid future headaches related to permissions and versions.