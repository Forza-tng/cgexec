# Linux Control Groups

Linux Control Groups, commonly referred to as cgroups, is a feature in the Linux kernel that provides a way to organise and manage system resources for processes. It allows you to allocate and limit resources like CPU, memory, and I/O bandwidth among different processes or groups of processes.

There are two versions of cgroups, the original version 1 and the `unified`, version 2.

# cgexec

cgexec is a Bash script that allows users to execute a command within a cgroup, providing control over resource limits such as CPU, I/O, and memory.

The script requires v2/unified cgroups to function.

Documentation is available at <https://wiki.tnonline.net/w/Linux/cgexec>.

## Usage

```
Attaches a program <cmd> to a unified cgroup with defined limits.

Usage: cgexec [options] <cmd> [cmd args]
Options:
 -c cpu.weight   (0-10000)  Set CPU priority
 -C cpu.max      (1-100)    Set max CPU time in percent
 -i io.weight    (1-10000)  Set I/O weight
 -m memory.high  (0-max)    Set soft memory limit
 -M memory.max   (0-max)    Set hard memory limit
 -g group        Create or attach to existing cgroup. Default is to use an ephemeral group
 -b path         Use <path> as cgroup root
 
Requires Linux Control Groups v2 mounted at <path>.
```

## Examples

Execute a command with default settings
```
cgexec echo "Hello, cgroups!"
```

Limit CPU and memory for a command
```
cgexec -c 50 -m 1G my_command
```

Attach to an existing cgroup
```
 cgexec -g mygroup my_command
```

Use a custom cgroup root
```
cgexec -b /sys/fs/cgroup/mysubsystem my_command
```

# Enable user cgroups
It is possible to allow normal users to create their own cgroups by creating an initial cgroup container that is owned by the user.

First set up a container for the user
```
mkdir -p /sys/fs/cgroup/user/forza/main
echo "+cpu +memory +io" > /sys/fs/cgroup/user/cgroup.subtree_control
echo "+cpu +memory +io" > /sys/fs/cgroup/user/forza/cgroup.subtree_control
chown forza:forza /sys/fs/cgroup/user/forza
chown forza:forza /sys/fs/cgroup/user/forza/cgroup.procs
chown forza:forza /sys/fs/cgroup/user/forza/cgroup.threads
chown -R forza:forza /sys/fs/cgroup/user/forza/main
chmod 775 /sys/fs/cgroup/user/forza
chmod 775 /sys/fs/cgroup/user/forza/main
```
User `<forza>` can now create new nested cgroups in `/sys/fs/cgroup/user/forza` and attach processes to it.
The catch-22 of cgroups is that all programs by default reside in in the root `/sys/fs/cgroup/cgroup.procs`, and even though a user can write to its own cgroup, they can not remove themselves from the root cgroup.

One solution is to let the user start a `screen` or `tmux` session and then use root/sudo to move it to the user's cgroup.

Then as root, check what `pids` belong to the uaer session and add them to the user cgroup:
```
# ps af -u forza
5309 pts/0    S+     0:00  |               \_ screen
5310 ?        Ss     0:00  |                   \_ SCREEN
5311 pts/1    Ss     0:00  |                       \_ -/bin/bash
5483 pts/1    R+     0:00  |                           \_ ps af -u forza

# echo 5309 > /sys/fs/cgroup/user/forza/main/cgroup.procs
# echo 5310 > /sys/fs/cgroup/user/forza/main/cgroup.procs
# echo 5311 > /sys/fs/cgroup/user/forza/main/cgroup.procs
```
Now, the user's screen session is in the `main` cgroup and the user can create additional cgroups and move processes started from within this session to that cgroup.

## Helper utility
The `user_cgroup` helper utility can automate the process of setting of user cgroups.

- `user_cgroup.initd` is an OpenRC init script that creates a cgroup hierarchy for specified users.
- `user_cgroup.confd` is the configuration file for OpenRC.
- `user_cgroup.sudoersd` is a sudo config file for allowing users to use the `user_cgroup` utility.
- `user_cgroup` small tool to move a uaer's shell to the user cgroup.

The sudo file is optional, but allows a user to put the script in `~/.bash_profile` for automatic use.

Once installed, a user can simply run `user_cgroup` to move itselslf to the user cgroup. Now it is possible for the user to create its own sub cgroups with `cgexec`.

```
# ./cgexec ls -l
* Creating cgroup: /sys/fs/cgroup/user/forza/cmd-GEP0
* Executing: ls -l

total 60
-rwxr-xr-x 1 forza forza  4402 Dec 17 14:20 cgexec
-rw-r--r-- 1 forza forza 34580 Dec 16 19:02 LICENSE
drwxr-xr-x 1 forza forza    72 Dec 16 21:16 openrc
-rw-r--r-- 1 forza forza  4106 Dec 17 13:57 README.md
-rwxr-xr-x 1 forza forza   315 Dec 17 13:37 user_cgroup
-rw-r--r-- 1 forza forza    79 Dec 17 13:40 user_cgroup.sudoersd

* Cleaning up cgroup /sys/fs/cgroup/user/forza/cmd-GEP0
```
