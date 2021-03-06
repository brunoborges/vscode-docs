---
Order: 5
Area: remote
TOCTitle: Tips and Tricks
PageTitle: Visual Studio Code Remote Development Troubleshooting Tips and Tricks
ContentId: 42e65445-fb3b-4561-8730-bbd19769a160
MetaDescription: Visual Studio Code Remote Development troubleshooting tips and tricks for SSH, Containers, and the Windows Subsystem for Linux (WSL)
DateApproved: 5/2/2019
---
# Remote Development Tips and Tricks

❗ **Note:** The **[Remote Development extensions](https://aka.ms/vscode-remote/download)** require **[Visual Studio Code Insiders](http://code.visualstudio.com/insiders)**.

---

This article covers troubleshooting tips and tricks for each of the Visual Studio Code [Remote Development](https://aka.ms/vscode-remote/download/extension) extensions. See the [SSH](/docs/remote/ssh.md), [Containers](/docs/remote/containers.md), and [WSL](/docs/remote/wsl.md) articles for details on setting up and working with each specific extension.

## SSH tips

### Configuring key based authentication

[SSH public key authentication](https://www.ssh.com/ssh/public-key-authentication) is a convenient, high security authentication method that combines a local "private" key with a "public" key that you associate with your user account on an SSH host. This section will walk you through how to generate these keys and add them to a host.

> **Tip:** PuTTY for Windows is not a [supported client](#installing-a-supported-ssh-client), but you can [convert your PuTTYGen keys](#reusing-a-key-generated-in-puttygen).

### Quick start: SSH key

To set up SSH key based authentication for your remote host:

1. Check to see if you already have an SSH key. The public key is typically located at `~/.ssh/id_rsa.pub` on macOS / Linux, and at `%USERPROFILE%\.ssh\id_rsa.pub` on Windows.

    If you do not have a key, run the following command in a terminal / command prompt to generate an SSH key pair:

    ```bash
    ssh-keygen -t rsa -b 4096
    ```

    > **Tip:** Don't have `ssh-keygen`? Install [a supported SSH client](#installing-a-supported-ssh-client).

2. Add the contents of your **local** public key (the `id_rsa.pub` file) to the appropriate `authorized_keys` file(s) on the remote host.

    On **macOS / Linux**, run the following command in a **local terminal**, replacing the user and host name as appropriate.

    ```bash
    ssh-copy-id your-user-name-on-host@host-fqdn-or-ip-goes-here
    ```

    On **Windows**, run the following commands in a **local command prompt** replacing the value of `REMOTEHOST` as appropriate.

    ```bat
    SET REMOTEHOST=your-user-name-on-host@host-fqdn-or-ip-goes-here

    scp %USERPROFILE%\.ssh\id_rsa.pub %REMOTEHOST%:~/tmp.pub
    ssh %REMOTEHOST% "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat ~/tmp.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm -f ~/tmp.pub"
    ```

### Improving your security with a dedicated key

While using a single SSH key across all your SSH hosts can be convenient, if anyone gains access to your private key, they will have access to all of your hosts as well. You can prevent this by creating a separate SSH key for your development hosts. Just follow these steps:

1. Generate a separate SSH key in a different file.

    On macOS / Linux, run the following command in a **local terminal**:

    ```bash
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa-remote-ssh
    ```

    On Windows, run the following command in a **local command prompt**:

    ```bat
    ssh-keygen -t rsa -b 4096 -f %USERPROFILE%\.ssh\id_rsa-remote-ssh
    ```

2. In VS Code, run **Remote-SSH: Open Configuration File...** in the Command Palette (`kbstyle(F1)`), select an SSH config file, and add (or modify) a host entry as follows:

    ```yaml
    Host name-of-ssh-host-here
        User your-user-name-on-host
        HostName host-fqdn-or-ip-goes-here
        IdentityFile ~/.ssh/id_rsa-remote-ssh
    ```

3. Add the contents of the **local** `id_rsa-remote-ssh.pub` file generated in step 1 to the appropriate `authorized_keys` file(s) on the remote host.

    On **macOS / Linux**, run the following command in a **local terminal**, replacing `name-of-ssh-host-here` with the host name in the SSH config file from step 2:

    ```bash
    ssh-copy-id -i ~/.ssh/id_rsa-remote-ssh.pub name-of-ssh-host-here
    ```

    On **Windows**, run the following commands in a **local command prompt**, replacing `name-of-ssh-host-here` with the host name in the SSH config file from step 2.

    ```bat
    SET REMOTEHOST=name-of-ssh-host-here
    SET PATHOFIDENTITYFILE=%USERPROFILE%\.ssh\id_rsa-remote-ssh.pub

    scp %PATHOFIDENTITYFILE% %REMOTEHOST%:~/tmp.pub
    ssh %REMOTEHOST% "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat ~/tmp.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm -f ~/tmp.pub"
    ```

### Reusing a key generated in PuTTYGen

If you used PuTTYGen to set up SSH public key authentication for the host you are connecting to, you need to convert your private key so that other SSH clients can use it. To do this:

1. Open PuTTYGen and load the private key you want to convert.
2. Select **Conversions > Export OpenSSH key** from the application menu. Save the converted key to a location such as `%USERPROFILE%\.ssh`.
3. Validate that the permissions on the exported key file only grant `Full Control` to your user, Administrators, and SYSTEM.
4. In VS Code, run **Remote-SSH: Open Configuration File...** in the Command Palette (`kbstyle(F1)`), select the SSH config file you wish to change, and add (or modify) a host entry in the config file as follows:

    ```yaml
    Host name-of-ssh-host-here
        User your-user-name-on-host
        HostName host-fqdn-or-ip-goes-here
        IdentityFile C:\path\to\your\exported\private\keyfile
    ```

### Troubleshooting hanging connections

If you are running into problems with VS Code hanging while trying to connect (and potentially timing out), there are a few things you can do to try to resolve the issue.

First, enable the `remote.SSH.showLoginTerminal` [setting](/docs/getstarted/settings.md) in VS Code and retry. If you are prompted to input a password or token, see [Enabling alternate SSH authentication methods](#enabling-alternate-ssh-authentication-methods) for details on reducing the frequency of prompts.

If this is not the problem, you likely are running into an issue with your SSH configuration. To troubleshoot, open the `Remote - SSH` category in the output window.

* If you see errors about permissions or an unprotected key, see [Fixing SSH file permission errors](#fixing-ssh-file-permission-errors).

* If you see `open failed: administratively prohibited: open failed`:
  1. Open `/etc/ssh/sshd_config` in an editor (like vim, nano, or pico).
  2. Add the setting  `AllowTcpForwarding yes`.
  3. Restart the SSH server (on Ubuntu, run `sudo systemctl restart sshd`).
  4. Retry.

You may also see other errors in the log, which can give hints as to what is going wrong or provide configuration recommendations.

### Enabling alternate SSH authentication methods

If you are connecting to an SSH remote host and are either:

- connecting with two-factor authentication,
- using password authentication,
- using an SSH key with a passphrase when the [SSH Agent](#setting-up-the-ssh-agent) is not running or accessible,

...you need to enable the `remote.SSH.showLoginTerminal` [setting](/docs/getstarted/settings.md) in VS Code. This setting displays the terminal whenever VS Code runs an SSH command. You can then enter your auth code, password, or passphrase when the terminal appears.

To avoid reentering your connection information each time, you can enable the `ControlMaster` feature so that OpenSSH runs multiple SSH sessions over a single connection.

To enable `ControlMaster`:

1. Add an entry like this to your SSH config file:

    ```yaml
    Host *
        ControlMaster auto
        ControlPath  ~/.ssh/sockets/%r@%h-%p
        ControlPersist  600
    ```

2. Then run `mkdir -p ~/.ssh/sockets` to create the sockets folder.

With `ControlMaster` enabled, you will only have to enter your auth code/password/passphrase once.

### Setting up the SSH Agent

If you are connecting to an SSH host using a key with a passphrase, you should ensure that the [SSH Agent](https://www.ssh.com/ssh/agent) is running. VS Code will automatically add your key to the agent so you don't have to enter your passphrase every time you open a remote VS Code window.

To verify that the agent is running and is reachable from VS Code's environment, run `ssh-add -l` in the terminal of a local VS Code window. You should see a listing of the keys in the agent (or a message that it has no keys). If the agent is not running, follow these instructions to start it. After starting the agent, be sure to restart VS Code.

**Windows:**

To enable SSH Agent automatically on Windows, start PowerShell as an Administrator and run the following commands:

```powershell
# Make sure you're running as an Administrator
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
Get-Service ssh-agent
```

Now the agent will be started automatically on login.

**Linux:**

To start the SSH Agent in the background, run:

```bash
eval "$(ssh-agent -s)"
```

To start the SSH Agent automatically on login, add these lines to your `~/.bash_profile`:

```bash
if [ -z "$SSH_AUTH_SOCK" ]
then
   # Check for a currently running instance of the agent
   RUNNING_AGENT="`ps -ax | grep 'ssh-agent -s' | grep -v grep | wc -l | tr -d '[:space:]'`"
   if [ "$RUNNING_AGENT" = "0" ]
   then
        # Launch a new instance of the agent
        ssh-agent -s &> .ssh/ssh-agent
   fi
   eval `cat .ssh/ssh-agent`
fi
```

**macOS:**

The agent should be running by default on macOS.

### Fixing SSH file permission errors

SSH can be strict about file permissions and if they are set incorrectly, you may see errors such as "WARNING: UNPROTECTED PRIVATE KEY FILE!". There are several ways to update file permissions in order to fix this, which are described in the sections below.

### Local SSH file and folder permissions

**macOS / Linux:**

On your local machine, make sure the following permissions are set:

| Folder / File |  Permissions |
|---------------|---------------------------|
| `.ssh` in your user folder | `chmod 700 ~/.ssh` |
| `.ssh/config` in your user folder | `chmod 600 ~/.ssh/config` |
| `.ssh/id_rsa.pub` in your user folder | `chmod 600 ~/.ssh/id_rsa.pub` |
| Any other key file | `chmod 600 /path/to/key/file` |

**Windows:**

The specific expected permissions can vary depending on the exact SSH implementation you are using. We strongly recommend using the out of box [Windows 10 OpenSSH Client](https://docs.microsoft.com/windows-server/administration/openssh/). If you are using this official client, cut-and-paste the following in an **administrator PowerShell window** to try to repair your permissions:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process

Install-Module -Force OpenSSHUtils -Scope AllUsers

Repair-UserSshConfigPermission ~/.ssh/config
Get-ChildItem ~\.ssh\* -Include "id_rsa","id_dsa" -ErrorAction SilentlyContinue | % {
    Repair-UserKeyPermission -FilePath $_.FullName @psBoundParameters
}
```

For all other clients, consult **your client's documentation** for what the implementation expects. However, note that not all SSH clients may work.

### Server SSH file and folder permissions

On the remote machine you are connecting to, make sure the following permissions are set:

| Folder / File | Linux / macOS Permissions |
|---------------|---------------------------|
| `.ssh` in your user folder on the server | `chmod 700 ~/.ssh` |
| `.ssh/authorized_keys` in your user folder on the server  | `chmod 600 ~/.ssh/authorized_keys` |

Note that only Linux hosts are currently supported, which is why permissions for macOS and Windows 10 have been omitted.

### Installing a supported SSH client

| OS | Instructions |
|----|--------------|
| Windows 10 / Server 2016 | Install the [Windows OpenSSH Client](https://docs.microsoft.com/windows-server/administration/openssh/openssh_install_firstuse). |
| Earlier Windows | Install [Git for Windows](https://git-scm.com/download/win) and select the **Use Git and optional Unix tools from the Command Prompt** option or manually add `C:\Program Files\Git\usr\bin` into your PATH. |
| macOS | Comes pre-installed. |
| Debian/Ubuntu | Run `sudo apt-get install openssh-client` |
| RHEL / Fedora / CentOS | Run `sudo yum install openssh-clients` |

### Installing a supported SSH server

| OS | Instructions | Details |
|----|--------------|---|
| Debian / Ubuntu | Run `sudo apt-get install openssh-server` |  See the [Ubuntu SSH](https://help.ubuntu.com/community/SSH?action=show) documentation for additional setup instructions. |
| RHEL / Fedora / CentOS | Run `sudo yum install openssh-server && sudo systemctl start sshd.service && sudo systemctl enable sshd.service` | You may need to omit `sudo` when running in a container. |
| Windows | Not supported yet. | |
| macOS | Not supported yet. | |

### Resolving hangs when doing a Git push or sync on an SSH host

If you clone a Git repository using SSH and your SSH key has a passphrase, VS Code's pull and sync features may hang when running remotely.

Either use an SSH key without a passphrase, clone using HTTPS, or run `git push` from the command line to work around the issue.

## Container tips

### Docker Desktop for Windows tips

[Docker Desktop](https://www.docker.com/products/docker-desktop) for Windows works well in most setups, but there are a few "gotchas" that can cause problems. Here are some tips on avoiding them:

1. **Use an AD domain account or local administrator account when sharing drives. Do not use an AAD (email-based) account.** AAD (email-based) accounts have well known issues, as documented in Docker issues [#132](https://github.com/docker/for-win/issues/132) and [#1352](https://github.com/docker/for-win/issues/1352). If you must use an AAD account, create a separate local administrator account on your machine that you use purely for the purpose of sharing drives. Follow  the [steps in this blog post](https://blogs.msdn.microsoft.com/stevelasker/2016/06/14/configuring-docker-for-windows-volumes/) to get everything set up.

2. **Stick with alphanumeric passwords to avoid drive sharing problems.** When asked to share your drives on Windows, you will be prompted for the username and password of an account with admin privileges on the machine. If you are warned about an incorrect username or password, this may be due to special characters in the password. For example, `!`, `[` and `]` are known to cause issues. Change your password to alphanumeric characters to resolve. See this issue about [Docker volume mounting problems](https://github.com/moby/moby/issues/23992#issuecomment-234979036) for details.

3. **Use your Docker ID to sign into Docker (not your email).** The Docker CLI only supports using your Docker ID, so using your email can cause problems. See Docker issue [#935](https://github.com/docker/hub-feedback/issues/935#issuecomment-300361781) for details.

If you are still having trouble, see the [Docker Desktop for Windows troubleshooting guide](https://docs.docker.com/docker-for-windows/troubleshoot/#volumes).

### Enabling file sharing in Docker Desktop

The VS Code [Remote - Containers](https://aka.ms/vscode-remote/download/containers) extension can only automatically mount your source code into a container if your code is in a folder or drive shared with Docker. If you open a dev container from a non-shared location, the container will successfully start but the workspace will be empty.

To change Docker's drive and folder sharing settings:

**Windows:**

1. Right-click on the Docker task bar item and select **Settings**.
2. Go to the **Shared Drives** tab and check the drive(s) where your source code is located.

**macOS:**

1. Click on the Docker menu bar item and select **Preferences**.
2. Go to the **File Sharing** tab. Confirm that the folder containing your source code is under one of the shared folders listed.

### Resolving Git line ending issues in containers (resulting in many modified files)

Since Windows and Linux use different default line endings, you may see files that appear modified but seem to have no differences aside from the line endings. To prevent this from happening, you can disable automatic line ending conversion and optionally add a `.gitattributes` file to your folder.

First run:

```bash
git config --global core.autocrlf false
```

This will disable automated conversation. If you would prefer to still always upload Unix-style line endings (LF), you can use the `input` option instead.

```bash
git config --global core.autocrlf input
```

Next, you can prevent others from facing this issue, regardless of their setting, by adding or modifying a  `.gitattributes` file in your repository.

For example, the `.gitattributes` settings below will force everything to be LF, except for Windows batch files that require CRLF:

```yaml
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```

You can add other file types in your repository that require CRLF to this same file.

Finally, reclone the repository so these settings take effect.

### Avoid setting up Git in a container when using Docker Compose

To avoid having to set up Git a second time in your container, VS Code automatically adds a volume mount to your local Git configuration when referencing an `image` or `Dockerfile`. The Docker Compose scenario gives you more control, but requires adding an extra configuration line to your `docker-compose.yml` file.

Specifically, add the following to the service you open in VS Code:

```yaml
volumes:
  # This lets you avoid setting up Git again in the container
  - ~/.gitconfig:/root/.gitconfig
```

If you do not have your email address set up locally, you may be prompted to do so. You can do this on your local machine by running the following command:

```bash
git config --global user.email "your.email@address"
```

If you prefer, you can extend your dev container configuration to achieve the same thing, without modifying your existing Docker Compose file. See [here for additional details](/docs/remote/containers.md#extending-your-docker-compose-file-for-development).

### Resolving hangs when doing a Git push or sync from a Container

If you clone a Git repository using SSH and your SSH key has a passphrase, VS Code's pull and sync features may hang when running remotely.

Either use an SSH key without a passphrase, clone using HTTPS, or run `git push` from the command line to work around the issue.

### Resolving errors about missing Linux dependencies

Some extensions rely on libraries not found in the certain Docker images. See the [Containers](/docs/remote/containers.md#installing-additional-software-in-the-sandbox) article for a few options on resolving this issue.

### Speeding up containers in Docker Desktop

By default, Docker Desktop only gives containers a fraction of your machine capacity. In most cases, this is enough, but if you are doing something that requires more capacity, you can increase memory, CPU, or disk use.

First, try [stopping any running containers](/docs/remote/containers.md#managing-containers) you are no longer using.

If this doesn't solve your problem, you may want to see if CPU usage is actually the issue or if there is something else going on. An easy way to check this is to install the [Resource Monitor extension](https://marketplace.visualstudio.com/items?itemName=mutantdino.resourcemonitor&ssr=false#overview). When installed in a container, it provides information about capacity for your containers in the Status bar.

![Resource use status bar](images/troubleshooting/resource-monitor.png)

If you'd like this extension to always be installed, add this to your `settings.json`:

```json
"remote.containers.defaultExtensions": [
    "mutantdino.resourcemonitor"
]
```

If you determine that you need to give your container more of your machine's capacity, follow these steps:

1. Right-click on the Docker task bar item and select **Settings** (**Preferences** on macOS).
2. Go to **Advanced** to increase CPU, Memory, or Swap.
3. Go to **Disk** to increase the amount of disk Docker is allowed to consume on your machine.

Finally, if your container is disk intensive, you should avoid using a volume mount of your local filesystem to store data files (for example database data files) particularly on Windows. Update your application's settings to use a folder inside the container instead.

### Cleaning out unused containers and images

If you see an error from Docker reporting that you are out of disk space, you can typically resolve this by cleaning out unused containers and images. There are a few ways to do this:

**Option 1: Use the Docker extension.**

 1. Install the [Docker extension](https://marketplace.visualstudio.com/items?itemName=PeterJausovec.vscode-docker) from the Extensions view if not already present.

    > **Note:** Using the Docker extension from a VS Code window opened in a container has some limitations. Most containers do not have the Docker command line installed. Therefore commands invoked from the Docker extension that rely on the Docker command line, for example **Docker: Show Logs**, fail. If you need to execute these commands, open a new local window and use the Docker extension from this VS Code window or [set up Docker inside your container](https://aka.ms/vscode-remote/samples/docker-in-docker).

 1. You can then go to the Docker view and expand the **Containers** or **Images** node, right-click, and select **Remove Container / Image**.

     ![Docker Explorer screenshot](images/containers/docker-remove.png)

**Option 2: Use the Docker CLI to pick containers to delete**:

1. Open a **local** terminal/command prompt (or use a local window in VS Code).
2. Type `docker ps -a` to see a list of all containers.
3. Type `docker rm <Container ID>` from this list to remove a container.
4. Type `docker image prune` to remove any unused images.

**Option 3: Use Docker Compose**:

1. Open a **local** terminal/command prompt (or use a local window in VS Code).
2. Go to the directory with your `docker-compose.yml` file.
3. Type `docker-compose down` to stop and delete the containers. If you have more than one Docker Compose file, you can specify additional Docker Compose files with the `-f` argument.

**Option 4: Delete all containers and images that are not running:**

1. Open a **local** terminal/command prompt (or use a local window in VS Code).
2. Type `docker system prune --all`.

### Adding another volume mount

You can add a volume mount to any local folder using these steps:

1. Configure the volume mount:

   - When an **image** or **Dockerfile** is referenced in `devcontainer.json`, add the following to the `runArgs` property in this same file:

        ```json
        "runArgs": ["-v","/local/source/path/goes/here:/target/path/in/container/goes/here"]
        ```

   - When a **Docker Compose** file is referenced, add the following to your `docker-compose.yml`:

        ```json
        volumes:
          - /local/source/path/goes/here:/target/path/in/container/goes/here
        ```

2. If you've already built the container and connected to it, run **Remote-Containers: Rebuild Container** from the Command Palette (`kbstyle(F1)`) to pick up the change.

### Connecting to multiple containers

Currently you can only connect to one container per VS Code window. However, you can spin up multiple containers and [attach to them](/docs/remote/containers.md#attaching-to-running-containers) from different VS Code windows to work around this limitation.

### Using Docker / Kubernetes from inside a dev container

You can use Docker and Kubernetes related CLIs and extensions from inside your development container by forwarding the Docker socket and installing the Docker CLI (and kubectl for Kubernetes) in the container. See the [Docker-in-Docker](https://aka.ms/vscode-remote/samples/docker-in-docker), [Docker-in-Docker Compose](https://aka.ms/vscode-remote/samples/docker-in-docker-compose), and [Kubernetes-Helm](https://aka.ms/vscode-remote/samples/kubernetes-helm) dev container definitions for details.

### Adding a non-root user to your dev container

Many images run as a root user by default. However, some provide one or more non-root users, that you can optionally use instead. If your image or Dockerfile provides a non-root user (but still defaults to root), you can opt into using it in one of two ways:

- When referencing an **image** or **Dockerfile**, add the following to your `devcontainer.json`:

    ```json
    "runArgs": ["-u", "user-name-goes-here"]
    ```

- If you are using **Docker Compose**, add the following to your service in `docker-compose.yml`:

    ```yaml
    user: user-name-goes-here
    ```

For images that only provide a root user, you can automatically create a non-root user by using a Dockerfile. For example, this snippet will create a user called `user-name-goes-here`, give it the ability to use `sudo`, and set it as the default:

```Dockerfile
ARG USERNAME=user-name-goes-here
RUN useradd -m $USERNAME
ENV HOME /home/$USERNAME

# [Optional] Add sudo support
RUN apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME && \
    chmod 0440 /etc/sudoers.d/$USERNAME

# ** Anything else you want to do like clean up goes here **

# [Optional] Set the default user
USER $USERNAME
```

### Resolving Dockerfile build failures for images using Debian 8

When building containers that use images based on Debian 8/Jessie — such as older versions of the `node:8` image — you may encounter the following error:

```text
...
Get:5 http://security.debian.org jessie/updates/main amd64 Packages [825 kB]
Get:6 http://deb.debian.org jessie/main amd64 Packages [9098 kB]
Fetched 10.1 MB in 6s (1458 kB/s)
W: Failed to fetch http://deb.debian.org/debian/dists/jessie-updates/InRelease  Unable to find expected entry 'main/binary-amd64/Packages' in Release file (Wrong sources.list entry or malformed file)
E: Some index files failed to download. They have been ignored, or old ones used instead.
The command '/bin/sh -c apt-get update     && apt-get -y install --no-install-recommends apt-utils 2>&1' returned a non-zero code: 100
Failed: Building an image from the Dockerfile.
```

This is a [well known issue](https://github.com/debuerreotype/docker-debian-artifacts/issues/66) caused by the Debian 8 being "archived". More recent versions of images typically resolve this problem, often by upgrading to Debian 9/Stretch.

There are two ways to resolve this error:

- **Option 1**: Remove any containers that depend on the image, remove the image, and then try building again. This should download an updated image that is not affected by the problem. See [cleaning out unused containers and images](#cleaning-out-unused-containers-and-images) for details.

- **Option 2**: If you don't want to delete your containers or images, add this line into your Dockerfile before any `apt` or `apt-get` command. It adds the needed source lists for Jessie:

    ```Dockerfile
    # Add archived sources to source list if base image uses Debian 8 / Jessie
    RUN cat /etc/*-release | grep -q jessie && printf "deb http://archive.debian.org/debian/ jessie main\ndeb-src http://archive.debian.org/debian/ jessie main\ndeb http://security.debian.org jessie/updates main\ndeb-src http://security.debian.org jessie/updates main" > /etc/apt/sources.list
    ```

### Other common Docker related errors and issues

This section explains how to work around common issues related to using Docker.

### Sign in errors to Docker Hub when an email is use

The Docker CLI only supports using your Docker ID, so using your email to sign in can cause problems. See Docker issue [#935](https://github.com/docker/hub-feedback/issues/935#issuecomment-300361781) for details.

As a workaround, use your Docker ID to sign into Docker rather than your email.

### High CPU utilization of Hyperkit on macOS

There is [known issue with Docker for Mac](https://github.com/docker/for-mac/issues/1759) that can drive high CPU spikes. In particular, high CPU usage occurring when watching files and building. If you see high CPU usage for `com.docker.hyperkit` in Activity Monitor while very little is going on in your dev container, you are likely hitting this issue. Follow the [Docker issue](https://github.com/docker/for-mac/issues/1759) for updates and fixes.

### debconf: delaying package configuration, since apt-utils is not installed

This error can typically be safely ignored and is tricky to get rid of completely. However, you can reduce it to one message in stdout when installing the needed package by adding the following to your Dockerfile:

```Dockerfile
# Configure apt
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils 2>&1

## YOUR DOCKERFILE CONTENT GOES HERE

ENV DEBIAN_FRONTEND=dialog
```

### Warning: apt-key output should not be parsed (stdout is not a terminal)

This non-critical warning tells you not to parse the output of `apt-key`, so as long as your script doesn't, there's no problem. You can safely ignore it.

This occurs in Dockerfiles because the `apt-key` command is not running from a terminal. Unfortunately, this error cannot be eliminated completely, but can be hidden unless the `apt-key` command returns a non-zero exit code (indicating a failure).

For example:

```Dockerfile
# (OUT=$(apt-key add - 2>&1) || echo $OUT) will only print the output with non-zero exit code is hit
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | (OUT=$(apt-key add - 2>&1) || echo $OUT)
```

You can also set the `APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE` environment variable to suppress the warning, but it looks a bit scary so be sure to add comments in your Dockerfile if you use it:

```Dockerfile
# Suppress an apt-key warning about standard out not being a terminal. Its use in this script is safe.
ENV APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=DontWarn
```

## WSL tips

### Selecting the distribution used by Remote - WSL

The [Remote - WSL](https://aka.ms/vscode-remote/download/wsl) extension uses your **default distribution**, which you can change using [wslconfig.exe](https://docs.microsoft.com/windows/wsl/wsl-config).

For example:

```bat
wslconfig /setdefault Ubuntu
```

You can see which distributions you have installed by running:

```bat
wslconfig /l
```

### Fixing problems with the code-insiders command not working

If typing `code-insiders` from a WSL terminal on Window does not work, you may be missing some key locations from your PATH in WSL.

Check by opening a WSL terminal and typing `echo $PATH`. You should see the following paths listed:

1. `/mnt/c/Windows/System32`
2. The VS Code Insiders install path. By default, this should be: `/mnt/c/Users/{username}/AppData/Local/Programs/Microsoft VS Code Insiders/bin`

If the VS Code Insiders install path is missing, edit your `.bashrc`, add the following, and start a new terminal:

```bash
WINDOWS_USER_ID=your-user-alias-here
export PATH=$PATH:/mnt/c/Windows/System32:/mnt/c/Users/$WINDOWS_USER_NAME/AppData/Local/Programs/Microsoft VS Code Insiders/bin
```

### Resolving errors about missing dependencies

Some extensions rely on libraries not found in the vanilla install of certain WSL Linux distributions. You can add additional libraries into your Linux distribution by using its package manager. For Ubuntu and Debian based distributions, run `sudo apt-get install <package>` to install the needed libraries. Check the documentation for your extension or the runtime that is mentioned for additional installation details.

### Resolving Git line ending issues in WSL (resulting in many modified files)

Since Windows and Linux use different default line endings, you may see files that appear modified but seem to have no differences aside from the line endings. To prevent this from happening, you can disable automatic line ending conversion and optionally add a `.gitattributes` file to your folder.

First run:

```bash
git config --global core.autocrlf false
```

This will disable automated conversation. If you would prefer to still always upload Unix-style line endings (LF), you can use the `input` option instead.

```bash
git config --global core.autocrlf input
```

Next, you can prevent others from facing this issue, regardless of their setting, by adding or modifying a  `.gitattributes` file in your repository.

For example, the `.gitattributes` settings below will force everything to be LF, except for Windows batch files that require CRLF:

```yaml
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```

You can add other file types in your repository that require CRLF to this same file.

Finally, reclone the repository so these settings take effect.

### Resolving hangs when doing a Git push or sync from WSL

If you clone a Git repository using SSH and your SSH key has a passphrase, VS Code's pull and sync features may hang when running remotely.

Either use an SSH key without a passphrase, clone using HTTPS, or run `git push` from the command line to work around the issue.

## Extension tips

While many extensions will work unmodified, there are a few issues that can prevent certain features from working as expected. In some cases, you can use another command to work around the issue, while in others, the extension may need to be modified. This section provides a quick reference for common issues and tips on resolving them. You can also refer to the main extension article on [Supporting Remote Development](/api/advanced-topics/remote-extensions) for an in-depth guide on modifying extensions to support remote extension hosts.

### Local absolute path settings fail when applied remotely

VS Code's local user settings are reused when you connect to a remote endpoint. While this keeps your user experience consistent, you may need to vary absolute path settings between your local machine and each host / container / WSL since the target locations are different.

Resolution: You can set endpoint-specific settings after you connect to a remote endpoint by running the **Preferences: Open Remote Settings** command from the Command Palette (`kbstyle(F1)`) or by clicking on the **Remote** tab in the settings editor. These settings will override any local settings you have in place whenever you connect.

### Browser does not open locally

Some extensions use external node modules or custom code to launch a browser window. Unfortunately, this may cause the extension to launch the browser remotely instead of locally.

Resolution: The extension can switch to using the `vscode.env.openExternal` API to resolve this problem. See the [extension guide](/api/advanced-topics/remote-extensions#opening-something-in-a-local-browser-or-application) for details.

### Clipboard does not work

Some extensions use node modules like `clipboardy` to integrate with the clipboard. Unfortunately, this may cause the extension to incorrectly integrate with the clipboard on the remote side.

Resolution: The extension can switch to the VS Code clipboard API to resolve the problem. See the [extension guide](/api/advanced-topics/remote-extensions#using-the-clipboard) for details.

### Cannot access local web server from browser or application

When working inside a container or SSH host, the port the browser is connecting to may be blocked.

Resolution: The extension can switch to the  `vscode.env.openExternal` API (which automatically forwards localhost ports) to resolve this problem. See the [extension guide](/api/advanced-topics/remote-extensions#opening-something-in-a-local-browser-or-application) for details.

### WebView contents do not appear

If the extension's WebView content uses an iframe to connect to a local web server, the port the WebView is connecting to may be blocked.

Resolution: The WebView API now includes a `portMapping` property that the extension can use to solve this problem. See the [extension guide](/api/advanced-topics/remote-extensions#accessing-localhost) for details.

### Blocked localhost ports

If you are trying to connect to a localhost port from an external application, the port may be blocked.

Resolution: There currently is no API for extensions to programmatically forward arbitrary ports, but you can use the **Remote-Containers: Forward Port from Container...** or **Remote-SSH: Forward Port from Active Host...** to do so manually.

### Errors storing extension data

Extensions may try to persist global data by looking for the `~/.config/Code` folder on Linux. This folder may not exist, which can cause the extension to throw errors like `ENOENT: no such file or directory, open '/root/.config/Code/User/filename-goes-here`.

Resolution: Extensions can use the `context.globalStoragePath` or `context.storagePath` property to resolve this problem. See the [extension guide](/api/advanced-topics/remote-extensions#persisting-extension-data-or-state) for details.

### Cannot sign in / have to sign in each time I connect to a new endpoint

Extensions that require sign in may persist secrets using their own code. This code can fail due to missing dependencies. Even if it succeeds, the secrets will be stored remotely, which means you have to sign in for every new endpoint.

Resolution: Extensions can use the `keytar` node module to solve this problem. See the [extension guide](/api/advanced-topics/remote-extensions#persisting-secrets) for details.

### Extensions that ship or acquire pre-built native modules fail

Native modules bundled with (or dynamically acquired for) a VS Code extension must be recompiled [using Electron's `electron-rebuild`](https://electronjs.org/docs/tutorial/using-native-node-modules). However, VS Code Server runs a standard (non-Electron) version of Node.js, which can cause binaries to fail when used remotely.

Resolution: Extensions need to be modified to solve this problem. They will need to include (or dynamically acquire) both sets of binaries (Electron and standard Node.js) for the "modules" version in Node.js that VS Code ships and then check to see if `context.executionContext === vscode.ExtensionExecutionContext.Remote` in their activation function to set up the correct binaries. See the [extension guide](/api/advanced-topics/remote-extensions#using-native-node.js-modules) for details.

### Extensions fail due to missing modules

Extensions that rely on Electron or VS Code base modules (not exposed by the extension API) without providing a fallback can fail when running remotely. You may see errors in the Developer Tools console like `original-fs` not being found.

Resolution: Remove the dependency on an Electron module or provide a fallback. See the [extension guide](/api/advanced-topics/remote-extensions#avoid-using-electron-modules) for details.

### Cannot access / transfer remote workspace files to local machines

Extensions that open workspace files in external applications may encounter errors because the external application cannot directly access the remote files.

Resolution: We are investigating options for how extensions might be able to transfer files from the remote workspace to solve this problem.

### Cannot access attached device from extension

Extensions that access locally attached devices will be unable to connect to them when running remotely.

Resolution: We are investigating the best approach to solve this problem.

## Questions and feedback

- See [Remote Development FAQ](/docs/remote/faq.md).
- Search on [Stack Overflow](https://stackoverflow.com/questions/tagged/vscode-remote).
- Add a [feature request](https://aka.ms/vscode-remote/feature-requests) or [report a problem](https://aka.ms/vscode-remote/issues/new).
