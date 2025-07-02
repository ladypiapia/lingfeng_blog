---
title: "Deploy a Python Application as a Stable Service with systemd"
date: 2025-07-02
draft: false
summary: ""
tags: ["Python","Linux"]
---
# How to Deploy a Python Application as a Stable Service on Linux with `systemd`

When running a Python script on a server, a common yet fragile method is to use `python3 my_app.py &`. While this approach runs the script in the background, it has a critical flaw: if the script crashes due to an error or the server reboots, the process is gone for good, requiring manual intervention to restart it. This is unacceptable for services that need to run 24/7.

To solve this, we can use `systemd`—the built-in system and service manager for modern Linux distributions like Ubuntu 16.04+ and CentOS 7+. `systemd` acts as a guardian for your application, enabling crucial features like **automatic startup, crash recovery (auto-restart), and centralized log management**. This makes it the gold standard for professional deployments.

This guide will walk you through creating a `systemd` service for your Python application, step by step.

### Step 1: Prepare Your Python Application

Before you begin, ensure that:
1.  Your Python script (e.g., `my_app.py`) runs correctly on the server.
2.  You know the **absolute paths** to your Python interpreter and your script.
    *   To find the absolute path to your Python interpreter: `which python3` (This might return `/usr/bin/python3`, `/usr/local/bin/python3`, etc.)
    *   Note the full directory path where your script is located.

### Step 2: Create the `systemd` Service File (.service)

`systemd` manages services through configuration files that end with `.service`. These files are typically stored in the `/etc/systemd/system/` directory.

1.  **Create the service file**. Give your service a descriptive name, like `my-python-app.service`. Use a text editor like `nano` or `vim` to create it:
    ```bash
    sudo nano /etc/systemd/system/my-python-app.service
    ```

2.  **Write the configuration**. Copy the template below into the file. We will break down what each line means.

    ```ini
    [Unit]
    Description=A description for your Python application
    After=network.target

    [Service]
    User=your_username
    Group=your_groupname
    WorkingDirectory=/path/to/your/project
    ExecStart=/path/to/your/python3 /path/to/your/project/my_app.py
    Restart=on-failure
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```

### Step 3: Understand the Configuration File

Let's dissect each section and directive of the configuration file:

*   **`[Unit]` Section**: Defines metadata and dependencies for the service.
    *   `Description`: A brief description of your service. This text is shown when you check the service status.
    *   `After=network.target`: Specifies that this service should start only after the network is available. This is essential for any application that needs network access (like a web service or an API).

*   **`[Service]` Section**: Defines the core behavior of the service.
    *   `User`: **(Important)** Specifies the user that will run the service. For security reasons, it is **highly recommended to use a non-root user**. Do not use `root` unless your application has a specific need for superuser privileges.
    *   `Group`: Specifies the group to run the service under (optional, but good practice).
    *   `WorkingDirectory`: Defines the script's **working directory**. This is crucial because any relative file paths in your Python code (like opening a config file or writing to a log) will be based on this directory.
    *   `ExecStart`: **(Core)** Defines the **full command** to start the service. You **must** use absolute paths for both the Python interpreter and your script.
    *   `Restart=on-failure`: This is the key to stability! It tells `systemd` to automatically restart the service if it exits with a non-zero status code (which usually indicates an error). You could also set it to `always` to restart it even if it's stopped manually.
    *   `RestartSec=5s`: Waits for 5 seconds before attempting to restart. This prevents a rapidly crashing script from overwhelming the system.

*   **`[Install]` Section**: Defines how the service is installed.
    *   `WantedBy=multi-user.target`: This means that when you `enable` the service, it will be configured to start automatically when the system boots into the standard multi-user mode.

### Step 4: A Practical Example - Deploying a Gemini UI Application

Now, let's apply this knowledge to the `gemini.service` configuration you provided. We'll name the service file `gemini-ui.service`.

**Configuration File** (`/etc/systemd/system/gemini-ui.service`):

```ini
[Unit]
Description=My Python Application
# Ensures the application starts after the network is ready
After=network.target

[Service]
# The user that runs the service
User=root
# The working directory of your Python project
WorkingDirectory=/root/codes/gemini-2.0-flash-exp-ui
# The command to start the service, using absolute paths!
# You can find the path to python3 with the `which python3` command
ExecStart=/usr/local/bin/python3 gemini_image_ui_base64.py 
# Restart the service on failure
Restart=on-failure 
# Time to wait before restarting
RestartSec=5s 

[Install]
WantedBy=multi-user.target
```

**Configuration Analysis**:
*   This is a standard and very practical configuration. It correctly sets the working directory and the start command.
*   `WorkingDirectory` is set to `/root/codes/gemini-2.0-flash-exp-ui`, which means any relative path operations within `gemini_image_ui_base64.py` will originate from this directory.
*   `ExecStart` uses the absolute path `/usr/local/bin/python3`, which is excellent practice.
*   **Security Note**: Running as `User=root` gives the application the highest privileges on the server. If the application has any security vulnerabilities, it could compromise the entire system. If the program doesn't require root privileges (e.g., it doesn't need to bind to ports below 1024), it's best practice to create and use a dedicated, unprivileged user for enhanced security.

### Step 5: Managing Your Service

After creating and saving your `.service` file, you can manage your service using the `systemctl` command.

1.  **Reload the `systemd` Configuration**:
    You must run this command every time you create or modify a `.service` file to make `systemd` aware of the changes.
    ```bash
    sudo systemctl daemon-reload
    ```

2.  **Start the Service**:
    ```bash
    sudo systemctl start gemini-ui.service
    ```

3.  **Check the Service Status (The Most Useful Command)**:
    This is your primary tool for troubleshooting! It shows whether the service is running, its process ID (PID), and the most recent log output.
    ```bash
    sudo systemctl status gemini-ui.service
    ```
    *   If you see `Active: active (running)`, congratulations, your service is running correctly!
    *   If you see `Active: failed`, the service failed to start. Check the logs below the status line to find out why.

4.  **Enable the Service to Start on Boot**:
    To make the service start automatically after a server reboot, you need to "enable" it.
    ```bash
    sudo systemctl enable gemini-ui.service
    ```
    (To disable it, use `sudo systemctl disable gemini-ui.service`)

5.  **Stop the Service**:
    ```bash
    sudo systemctl stop gemini-ui.service
    ```

6.  **Restart the Service**:
    Use this command after you've modified your Python code and want to apply the changes.
    ```bash
    sudo systemctl restart gemini-ui.service
    ```

### Step 6: Viewing Logs

`systemd` has its own powerful logging system called `journald`. You can use the `journalctl` command to easily view all output from your application (including `print` statements and error tracebacks).

*   **View All Logs for the Service**:
    ```bash
    sudo journalctl -u gemini-ui.service
    ```

*   **Tailing the Logs in Real-Time (like `tail -f`)**:
    This is extremely useful for debugging, as it shows new log entries as they appear.
    ```bash
    sudo journalctl -u gemini-ui.service -f
    ```

*   **View the Last 100 Log Entries**:
    ```bash
    sudo journalctl -u gemini-ui.service -n 100
    ```

---

### Conclusion

By following these steps, you have successfully transformed a simple Python script into a robust, reliable, and manageable system service. You no longer need to worry about crashes or server reboots taking your application offline. This is a critical step in moving from a "works-for-now" setup to a professional, production-ready deployment.