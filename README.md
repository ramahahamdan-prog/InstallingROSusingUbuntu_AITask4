# InstallingROSusingUbuntu_AITask4
## Installing ROS 2 (Humble) on Ubuntu 22.04 (Jammy)

## Overview
This document walks through the full process of installing **ROS 2 Humble Desktop** on an **Ubuntu 22.04 (Jammy Jellyfish)** system, from updating the base OS packages to adding the official ROS 2 APT repository, installing the desktop package, and finally verifying that the installation works correctly.

It also documents a real permission issue that came up while adding the ROS 2 repository key and source list, and how it was resolved.

---

## Features
- Full system update and upgrade (`apt update && apt upgrade`) before installation.
- Installation of required prerequisite tools (`software-properties-common`, `curl`).
- Import of the official ROS 2 GPG signing key using a dedicated keyring file (modern, secure method — no deprecated `apt-key`).
- Registration of the ROS 2 APT repository (`packages.ros.org/ros2/ubuntu`, `jammy main`).
- Installation of the full **ros-humble-desktop** metapackage (RViz2, rqt tools, ROS core/base libraries, and all their dependencies).
- Automatic sourcing of the ROS 2 environment (`setup.bash`) added to `~/.bashrc` so every new terminal session has ROS 2 ready to use.
- Verification of the installation using `ros2 -h` and `$ROS_DISTRO`.

---

## Technologies Used
- **Ubuntu 22.04 LTS (Jammy Jellyfish)**
- **ROS 2 Humble Hawksbill**
- **APT / dpkg** package management
- **GPG keyrings** for secure package signing
- **Bash** shell scripting (`.bashrc` sourcing)
- **curl** for downloading the ROS signing key

---

## How to Install

### 1. Update the system
```bash
sudo apt update && sudo apt upgrade
```
This refreshes all package lists from Ubuntu's official repositories (security, main, universe, multiverse, updates, backports) and upgrades any outdated packages.

![apt update output](/01-apt-update.png)

### 2. Install prerequisites
```bash
sudo apt install software-properties-common curl
```

### 3. Add the ROS 2 GPG key
```bash
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
```

### 4. Add the ROS 2 repository to APT sources
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

### 5. Refresh the package index
```bash
sudo apt update
```
This now picks up the new ROS 2 repository (`packages.ros.org`) alongside the standard Ubuntu ones.

### 6. Install ROS 2 Humble Desktop
```bash
sudo apt install ros-humble-desktop
```
This pulls in a large set of dependencies (build tools, Boost libraries, OpenCV-related packages, RViz2, rqt plugins, etc.) and installs the full desktop variant of ROS 2.

![ROS key setup and desktop install](/02-ros-key-and-install.png)

### 7. Source ROS 2 automatically in every terminal
```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 8. Verify the installation
```bash
ros2 -h
echo $ROS_DISTRO
```
`ros2 -h` should print the full list of available `ros2` subcommands, and `$ROS_DISTRO` should return `humble`.

![Verifying the ROS 2 installation](/03-ros2-verify.png)

---

## Problems Encountered & How I Fixed Them

### Problem 1: `-bash: /dev/null: Permission denied`
When adding the ROS 2 repository, running:
```bash
echo "deb [...] http://packages.ros.org/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```
failed with:
```
-bash: /dev/null: Permission denied
```

**Cause:** `sudo` only elevates privileges for the `tee` command itself, not for the shell's output redirection (`>`). The `>` redirection to `/dev/null` is handled by the *current, unprivileged* shell, so the shell — not `tee` — tries to open `/dev/null` and can, in some restricted/misconfigured shell environments, fail with a permission error.

**Fix:** I re-ran the exact same command again. On the successful attempt the pipeline completed without error, since `tee` (running as root via `sudo`) wrote the actual APT source file correctly and the `> /dev/null` redirection is just discarding `tee`'s stdout copy (a formality, not the real write). The important part — `sudo tee /etc/apt/sources.list.d/ros2.list` — succeeded and created the file with root permissions, which was confirmed in the next step when `sudo apt update` successfully fetched `InRelease` and `Packages` from `packages.ros.org`.

### Problem 2: `ros2 --version` / `ros2 -version` not recognized
Trying to check the ROS 2 version with:
```bash
ros2 --version
ros2 -version
```
returned errors:
```
ros2: error: unrecognized arguments: --version
invalid choice: '-version'
```

**Cause:** The `ros2` CLI tool does not support a `--version` or `-version` flag — this is a common assumption carried over from other CLI tools, but `ros2`'s argument parser doesn't implement it.

**Fix:** Instead of relying on a version flag, I used:
```bash
ros2 -h
echo $ROS_DISTRO
```
`echo $ROS_DISTRO` correctly returned `humble`, confirming the installed distribution, and `ros2 -h` confirmed the CLI was fully functional with all expected subcommands (`action`, `bag`, `daemon`, `launch`, `node`, `topic`, `run`, etc.).

---

## Challenges
- **Understanding privilege escalation with pipes and redirection:** The `Permission denied` error on `/dev/null` was confusing at first because `sudo` was clearly present in the command. It required understanding that `sudo` in a pipeline only applies to the command it directly precedes (`tee`), not to shell redirections performed by the parent shell.
- **Large dependency tree:** Installing `ros-humble-desktop` pulled in a very large number of additional packages (build tools, Boost libraries 1.74 in dev/runtime variants, Qt/GUI dependencies, `gcc`/`g++`, `cmake`, Java runtime, etc.), which made the install step slow and required patience and sufficient disk space.
- **CLI assumptions from other tools:** Expecting `--version` to work (as it does in most Linux CLI tools) didn't apply to `ros2`, which was a small but useful reminder to check a tool's actual `-h`/help output rather than assuming standard conventions.
- **Correctly persisting the environment:** Making sure ROS 2 is available in *every* new terminal session (not just the current one) required appending the `source /opt/ros/humble/setup.bash` line to `~/.bashrc` rather than just running `source` once in the active shell.

---

## Result
ROS 2 Humble Desktop was successfully installed and verified on Ubuntu 22.04, with the environment automatically sourced on every new terminal session (`$ROS_DISTRO` = `humble`).
