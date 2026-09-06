# Jenkins Docker Slave Pipeline

A Jenkins pipeline to verify whether the **Docker slave/agent configuration** is working as expected.

---

## 1. Prerequisites

Before running the pipeline, make sure **Git** is installed on the Jenkins server.

### Install Git

```bash
sudo dnf install git -y
```

Verify the installation:

```bash
git --version
```

---

## 2. Jenkins Plugins

Go to:

**Jenkins → Manage Jenkins → Plugins → Installed Plugins**

Install the following plugins:

* **Docker Pipeline**
* **Pipeline: Stage View**

---

# Issues & Troubleshooting

## Issue 1: Jenkins Built-in Node Keeps Going Offline

### Problem

The Jenkins built-in node was manually brought **Online**, but it immediately went **Offline** again.

### Step 1: Check Jenkins Logs

After bringing the node online, run:

```bash
sudo journalctl -u jenkins --since "10 minutes ago" --no-pager
```

The logs showed that the node was going offline because of a **disk space threshold issue**.

### Step 2: Check Jenkins Node Monitor Threshold

Check the Jenkins node monitor configuration:

```bash
sudo cat /var/lib/jenkins/nodeMonitors.xml
```

The configured disk-space threshold was approximately:

```text
1 GB
```

However, `/tmp` had less available space than the required threshold.

### Step 3: Check `/tmp` Disk Space

Check the available disk space:

```bash
df -h
```

Check how `/tmp` is mounted:

```bash
mount | grep ' /tmp '
```

Check `/tmp` configuration in `/etc/fstab`:

```bash
cat /etc/fstab | grep tmp
```

---

## Temporary Solution

Increase the `/tmp` size to **2 GB**:

```bash
sudo mount -o remount,size=2G /tmp
```

Verify the new size:

```bash
df -h
mount | grep ' /tmp '
```

Reload systemd and restart Jenkins:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

After this change, the Jenkins node came back online successfully.

### Why This Was Only a Temporary Solution

The remount configuration does **not survive an EC2 stop/start**.

```text
Current Boot
     │
     ▼
~457 MB /tmp
     │
     ▼
Remount to 2 GB
     │
     ▼
2 GB /tmp
     │
     ▼
Jenkins Node ONLINE
```

After EC2 **Stop → Start**:

```text
EC2 Stop/Start
     │
     ▼
/tmp configuration resets
     │
     ▼
~457 MB /tmp
     │
     ▼
Jenkins Node OFFLINE
```

Therefore, a permanent configuration was required.

---

# Permanent Solution

Do **not** modify the vendor-managed systemd file:

```text
/usr/lib/systemd/system/tmp.mount
```

Instead, create a **systemd drop-in override**.

This configuration is stored under `/etc/systemd`, which is part of the persistent root/EBS filesystem and therefore survives an EC2 stop/start.

---

## Step 1: Create the Override Directory

```bash
sudo mkdir -p /etc/systemd/system/tmp.mount.d
```

---

## Step 2: Create the Override Configuration

Create the configuration without opening an editor:

```bash
sudo tee /etc/systemd/system/tmp.mount.d/override.conf > /dev/null <<'EOF'
[Mount]
Options=mode=1777,strictatime,nosuid,nodev,size=1536M,nr_inodes=1m
EOF
```

This configures `/tmp` with a maximum size of:

**1536 MB = 1.5 GB**

This is above Jenkins' **1 GiB disk-space threshold**.

---

## Step 3: Verify the Override

Check the configuration file:

```bash
sudo cat /etc/systemd/system/tmp.mount.d/override.conf
```

Expected output:

```text
[Mount]
Options=mode=1777,strictatime,nosuid,nodev,size=1536M,nr_inodes=1m
```

Check the complete systemd configuration:

```bash
sudo systemctl cat tmp.mount
```

At the bottom, you should see:

```text
# /etc/systemd/system/tmp.mount.d/override.conf
[Mount]
Options=mode=1777,strictatime,nosuid,nodev,size=1536M,nr_inodes=1m
```

---

## Step 4: Reload Systemd

```bash
sudo systemctl daemon-reload
```

> **Note:** Do not immediately restart `tmp.mount` if `/tmp` is actively being used. Restarting the mount while it is in use may fail.

---

## Step 5: Apply the Configuration

For a clean application of the new `/tmp` mount configuration, reboot the EC2 instance during a suitable maintenance window:

```bash
sudo reboot
```

The configuration is stored at:

```text
/etc/systemd/system/tmp.mount.d/override.conf
```

Since this file is stored on the persistent root/EBS filesystem, it will **survive an EC2 stop/start**.

---

## Step 6: Verify `/tmp` After Reboot

After reconnecting to the EC2 instance:

```bash
df -h /tmp
```

Expected result:

```text
Filesystem    Size    Used    Avail    Use%    Mounted on
tmpfs         1.5G    ...     >1G      ...     /tmp
```

Also verify the mount:

```bash
mount | grep ' /tmp '
```

The output should contain:

```text
size=1572864k
```

or an equivalent value representing approximately **1.5 GB**.

### Result

The Jenkins built-in node disk-space issue is now **permanently resolved**.

---

# Issue 2: Jenkins User Does Not Have an Interactive Shell

## Problem

The Jenkins user existed on the server, but its login shell was configured as:

```text
/bin/false
```

Because of this, it was not possible to directly switch to the Jenkins user for interactive troubleshooting.

Commands such as the following had to be used:

```bash
sudo -u jenkins pwd
sudo -u jenkins whoami
```

---

## Step 1: Check Jenkins User Configuration

Run:

```bash
grep '^jenkins:' /etc/passwd
```

Initially, the output was:

```text
jenkins:x:993:993:Jenkins Automation Server:/var/lib/jenkins:/bin/false
```

The last field:

```text
/bin/false
```

means the Jenkins user does not have an interactive login shell.

---

## Step 2: Assign Bash Shell to Jenkins User

Run:

```bash
sudo usermod -s /bin/bash jenkins
```

This changes the Jenkins user's login shell from `/bin/false` to `/bin/bash`.

---

## Step 3: Verify the Change

Run:

```bash
grep '^jenkins:' /etc/passwd
```

Expected output:

```text
jenkins:x:993:993:Jenkins Automation Server:/var/lib/jenkins:/bin/bash
```

Now the Jenkins user has an interactive Bash shell.

You can switch to the Jenkins user using:

```bash
sudo su -s /bin/bash jenkins
```

Or:

```bash
sudo -iu jenkins
```

---

# Quick Troubleshooting Summary

| Issue                                 | Root Cause                                                     | Solution                                                   |
| ------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| Jenkins node keeps going offline      | `/tmp` available space was below Jenkins' disk-space threshold | Configure `/tmp` with a persistent 1.5 GB systemd override |
| `/tmp` size resets after EC2 restart  | Temporary remount was not persistent                           | Create `/etc/systemd/system/tmp.mount.d/override.conf`     |
| Jenkins user has no interactive shell | Shell configured as `/bin/false`                               | Change shell to `/bin/bash` using `usermod`                |
| Git not available                     | Git was not installed                                          | Install using `sudo dnf install git -y`                    |

---

# Final Configuration

After completing the above steps:

* Git is installed and available.
* Required Jenkins plugins are installed.
* Docker slave/agent pipeline can be tested.
* Jenkins built-in node remains online after the `/tmp` configuration is persisted.
* `/tmp` is configured with approximately **1.5 GB** capacity.
* Jenkins user has an interactive Bash shell for troubleshooting.
