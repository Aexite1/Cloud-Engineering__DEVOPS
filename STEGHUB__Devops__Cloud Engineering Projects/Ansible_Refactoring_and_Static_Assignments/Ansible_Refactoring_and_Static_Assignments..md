# Ansible Refactoring & Static Assignments (Imports and Roles)

This project continues the work in the `ansible-config-mgt` repository by refactoring Ansible code, creating static assignments, and using playbook imports. Playbook imports allow existing playbooks to be reused from a parent playbook, improving organization and reducing duplication.

## Step 1 - Jenkins job enhancement

The Jenkins job configuration is updated first. Each code change currently creates a separate build workspace, which can make it inconvenient to access the files from one central location. Retained build data can also consume storage on the Jenkins server. A second Jenkins project/job is therefore configured to collect files produced by the existing `ansible` job. This procedure requires the **Copy Artifact** plugin.

The primary method in this section retains the dedicated-directory approach: artifacts are copied into a dedicated directory on the `Jenkins-Ansible` server at `/home/ubuntu/ansible-config-artifact`.

### 1. Create the dedicated artifact directory

Connect to the `Jenkins-Ansible` server and create a directory called `ansible-config-artifact`. This is where the downstream Jenkins job will place the archived files after each build.

![Connect to the Jenkins-Ansible server](<./images/Screenshot 2026-09-23 173004.png>)

```bash
sudo mkdir -p /home/ubuntu/ansible-config-artifact
```

![Create the artifact directory](<./images/Screenshot 2026-09-23 173008.png>)

### 2. Configure ownership and permissions

Jenkins needs permission to write to the artifact directory. Avoid using `chmod -R 0777`: that setting gives every local user read, write, and execute access and is not necessary for this task.

First, identify the account under which the Jenkins service runs. On a typical Ubuntu package installation, this account is `jenkins`, but installations can use a different service account.

```bash
# Check the Jenkins service account
ps -eo user,comm | grep '[j]enkins'

# Check whether the expected account exists
id jenkins
```

If the Jenkins service uses a different account, replace `jenkins` in the commands below with the actual service account. The commands also assume that the Ubuntu login account is `ubuntu`.

Create a dedicated group for access to the artifact directory, then add the Ubuntu account to that group:

```bash
# Create the group if it does not already exist
getent group ansible-artifacts || sudo groupadd --system ansible-artifacts

# Allow the Ubuntu account to read the copied artifacts
sudo usermod -aG ansible-artifacts ubuntu
```

After creating the directory, make Jenkins its owner, and assign the dedicated group:

```bash
sudo mkdir -p /home/ubuntu/ansible-config-artifact

sudo chown jenkins:ansible-artifacts \
  /home/ubuntu/ansible-config-artifact

# Owner: read, write and traverse
# Group: read and traverse
# Others: no access
# The setgid bit makes new items inherit the directory group
sudo chmod 2750 /home/ubuntu/ansible-config-artifact
```

The Jenkins account must also be able to traverse the parent directory `/home/ubuntu`. It does not need permission to list or read the Ubuntu user's home directory. An access-control entry (ACL) can grant only the directory-traversal permission:

```bash
# Install ACL utilities if setfacl is not already available
command -v setfacl || sudo apt-get update
command -v setfacl || sudo apt-get install -y acl

# Allow Jenkins to pass through /home/ubuntu without listing its contents
sudo setfacl -m u:jenkins:--x /home/ubuntu
```

The dedicated group is then granted read and traversal access to the artifact directory. The default ACL makes the group access inherit when Jenkins creates new files and subdirectories:

```bash
sudo setfacl -m g:ansible-artifacts:r-x \
  /home/ubuntu/ansible-config-artifact

sudo setfacl -d -m g:ansible-artifacts:r-X \
  /home/ubuntu/ansible-config-artifact
```

The permissions have the following purpose:

- `2750` gives the directory owner full access, the group read and traversal access, and unrelated local users no access. The leading `2` sets the setgid bit so new items inherit the directory's group.
- `u:jenkins:--x` gives Jenkins traversal permission on `/home/ubuntu`. It does not grant Jenkins permission to list the home directory.
- `g:ansible-artifacts:r-x` lets members of the dedicated group read directory contents and traverse the directory.
- The default ACL applies group read/traversal permissions to new items. Files still need their normal read bit for group members; directories need execute permission for traversal.

If the artifact directory already contains files from an earlier attempt, correct their ownership and permissions as well:

```bash
sudo chown -R jenkins:ansible-artifacts \
  /home/ubuntu/ansible-config-artifact

sudo find /home/ubuntu/ansible-config-artifact \
  -type d -exec chmod 2750 {} +

sudo find /home/ubuntu/ansible-config-artifact \
  -type f -exec chmod 0640 {} +
```

Check the directory and ACL configuration:

```bash
namei -l /home/ubuntu/ansible-config-artifact
getfacl /home/ubuntu
getfacl /home/ubuntu/ansible-config-artifact
```

Test that Jenkins can write to the destination. This test runs as the Jenkins service account rather than relying on the permissions of the current terminal user:

```bash
sudo -u jenkins sh -c \
  'touch /home/ubuntu/ansible-config-artifact/.jenkins-write-test && rm /home/ubuntu/ansible-config-artifact/.jenkins-write-test'
```

After Jenkins copies the artifacts, the Ubuntu account can read them once the updated supplementary group membership has taken effect. A new login session is required after `usermod`. Group membership can be checked with:

```bash
groups ubuntu
```

The output is expected to include `ansible-artifacts`. The Ubuntu account is not added to the `jenkins` group; the dedicated group keeps artifact access separate from Jenkins' other files.

> **Environment note:** These commands assume an Ubuntu-based Jenkins-Ansible server, an account named `ubuntu`, a Jenkins service account named `jenkins`, and a filesystem that supports POSIX ACLs. Confirm the service account and install the ACL utilities before applying the commands. If the server uses another distribution or service account, adjust the package-installation and account names accordingly.

![Artifact directory permission configuration](<./images/chmod-artifact-dir.png>)

### 3. Install the Copy Artifact plugin

Go to the Jenkins web console:

`Manage Jenkins` → `Plugins` → `Available plugins`

The **Copy Artifact** plugin is installed from the available plugins list. Any restart recommendation displayed by Jenkins is followed, since restart requirements can vary by plugin version.

![Search for the Copy Artifact plugin](<./images/Screenshot 2026-09-23 190435.png>)

### 4. Configure the source `ansible` job to archive its files

The Copy Artifact plugin copies **archived artifacts** from a completed build. Installing the plugin does not automatically make every file in the source job's workspace available to another job.

Open the existing `ansible` Freestyle project and configure it to archive the repository files needed by the Ansible run.

In the source job's configuration, add the **Archive the artifacts** post-build action. A broad pattern that preserves the repository's directory structure is:

```text
**/*
```

If the repository's `.git` metadata is present in the workspace, exclude it from the archived files using the exclusion field:

```text
.git/**
```

The archived file list from a successful source build is checked for the directories required by the playbooks, such as `inventory`, `playbooks`, `static-assignments`, and `roles`. Secrets, private SSH keys, credentials, and other sensitive files are excluded. Secrets are stored in Jenkins Credentials or another approved secret-management system.

### 5. Create the downstream `save_artifacts` project

Create a new Freestyle project and name it `save_artifacts`.

![Create the save_artifacts project](<./images/Screenshot 2026-09-23 190635.png>)

Configure this project to run after the existing `ansible` project has completed. In a Freestyle job, use the **Build after other projects are built** trigger and enter `ansible` as the upstream project.

![Configure the upstream build trigger](<./images/Screenshot 2026-09-23 190831.png>)

**Note:** Build-discard policies can reduce storage consumption. For example, the jobs can be configured to retain the last two or five build results. The retention policy must not remove an upstream build before the downstream job has copied its artifacts.

### 6. Configure the downstream job to copy artifacts into the dedicated directory

The main purpose of the `save_artifacts` project is to copy the archived files from `ansible` into:

```text
/home/ubuntu/ansible-config-artifact
```

In the `save_artifacts` project, add the **Copy artifacts from another project** build step and configure it as follows:

- **Project name:** `ansible`
- **Which build:** select the stable upstream builds that triggered this project. This ensures that the downstream job copies the artifacts from the build that caused it to run, rather than accidentally copying a different, newer build.
- **Artifacts to copy:** use `**` or leave blank to copy all archived artifacts, or specify only the required paths.
> **Build selection:** The **Upstream build that triggered this project** option copies the exact upstream build that initiated the downstream job. This is more precise than selecting the latest stable build, which could refer to a different build.

- **Target directory:** `/home/ubuntu/ansible-config-artifact`

The target directory is an absolute path on the Jenkins agent/node where the copy operation runs. In this project, that should be the `Jenkins-Ansible` server. If the job runs on a different agent, the directory would be on that agent instead.

![Configure the Copy Artifact build step](<./images/Screenshot 2026-09-23 191445.png>)

On the source `ansible` job, configure the Copy Artifact permission setting to allow the `save_artifacts` job to copy its artifacts. Use the job's **Permission to Copy Artifact** setting and specify `save_artifacts`. A wildcard is avoided unless every matching job is explicitly authorized to retrieve the source artifacts. The exact location of this setting can vary with the Copy Artifact plugin version and Jenkins job type.

The plugin performs its own job-level permission checks. The filesystem permissions configured earlier are separate: they control whether the Jenkins service can write to the destination directory.

### 7. Test the two-job setup

The setup is tested by making a small change to the `README.md` file in the `ansible-config-mgt` repository and pushing it to the branch monitored by the Jenkins webhook. The configured source-job branch is used (for example, `main` or `master`).

![Edit the README file](<./images/Screenshot 2026-09-23 191521.png>)

For this exercise, the line `Testing jenkins build` is removed.

![Remove the test line from README](<./images/Screenshot 2026-09-23 191536.png>)

The webhook should trigger the `ansible` job. After that build completes, the `save_artifacts` job should run and copy its archived files into `/home/ubuntu/ansible-config-artifact`.

After both Jenkins jobs complete successfully, the destination directory is checked with:

```bash
ls -la /home/ubuntu/ansible-config-artifact
```

The directory listing is expected to show the archived project files and their directory structure. Subsequent source builds update the directory when they trigger the downstream job.

The Jenkins workflow provides a centralized location for the Ansible files, eliminating the need to locate an individual build workspace.

The following is an example of the permission error that can occur when Jenkins cannot create or write to the destination directory:

```text
Started by upstream project "ansible" build number 13
originally caused by:
 Started by GitHub push by Aexite1
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/save_artifacts
FATAL: /home/ubuntu/ansible-config-artifact
java.nio.file.AccessDeniedException: /home/ubuntu/ansible-config-artifact
    at java.base/sun.nio.fs.UnixException.translateToIOException(UnixException.java:90)
    at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:106)
    at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:111)
    at java.base/sun.nio.fs.UnixFileSystemProvider.createDirectory(UnixFileSystemProvider.java:397)
    at java.base/java.nio.file.Files.createDirectory(Files.java:700)
    at java.base/java.nio.file.Files.createAndCheckIsDirectory(Files.java:807)
    at java.base/java.nio.file.Files.createDirectories(Files.java:793)
    at hudson.FilePath.mkdirs(FilePath.java:3753)
    at hudson.FilePath$Mkdirs.invoke(FilePath.java:1419)
    at hudson.FilePath$Mkdirs.invoke(FilePath.java:1414)
    at hudson.FilePath.act(FilePath.java:1235)
    at hudson.FilePath.act(FilePath.java:1218)
    at hudson.FilePath.mkdirs(FilePath.java:1409)
    at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.copy(CopyArtifact.java:704)
    at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.perform(CopyArtifact.java:668)
    at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.perform(CopyArtifact.java:552)
    at jenkins.tasks.SimpleBuildStep.perform(SimpleBuildStep.java:123)
    at hudson.tasks.BuildStepCompatibilityLayer.perform(BuildStepCompatibilityLayer.java:80)
    at hudson.tasks.BuildStepMonitor$1.perform(BuildStepMonitor.java:20)
    at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuildExecution.java:818)
    at hudson.model.Build$BuildExecution.build(Build.java:199)
    at hudson.model.Build$BuildExecution.doRun(Build.java:164)
    at hudson.model.AbstractBuild$AbstractBuildExecution.run(AbstractBuild.java:526)
    at hudson.model.Run.execute(Run.java:1894)
    at hudson.model.FreeStyleBuild.run(FreeStyleBuild.java:44)
    at hudson.model.ResourceController.execute(ResourceController.java:101)
    at hudson.model.Executor.run(Executor.java:446)
Finished: FAILURE
```

The error above is from the Jenkins console output. It indicates that the process performing the copy could not create or access the destination directory. Setting `chmod -R 0777` is not a secure fix, and changing the Ubuntu user's group membership to `jenkins` does not give Jenkins access to the Ubuntu group. Check the Jenkins service account, parent-directory traversal permissions, destination ownership, and ACLs as described above.


Run the setup test again and confirm that the build completes successfully and that the files are present in the destination directory.

![Successful artifact copy build](<./images/Screenshot 2026-09-23 19244.jpg>)

### Alternative method - Copy artifacts into the downstream job workspace

The dedicated-directory method above is the primary method for this project. There is another option: the Copy Artifact plugin can copy the archived files directly into a directory inside the downstream job's workspace.

With this alternative, Jenkins does not need a separately created directory such as `/home/ubuntu/ansible-config-artifact`. The plugin creates the target directory and its parent directories inside the downstream workspace when they do not already exist.

For example, the downstream workspace could look like this:

```text
/var/lib/jenkins/workspace/save_artifacts/
└── artifacts/
    ├── inventory/
    ├── playbooks/
    ├── static-assignments/
    └── roles/
```

To configure this method:

1. Keep the source `ansible` job's **Archive the artifacts** post-build action. The source job must still archive the files before another job can retrieve them.
2. Open the downstream `save_artifacts` job and add **Copy artifacts from another project**.
3. Set the source project to `ansible`.
4. Select the upstream build that triggered the downstream job.
5. Set the target directory to `artifacts/` rather than an absolute server path.
6. Restrict the source job's **Permission to Copy Artifact** setting to the authorized downstream job, such as `save_artifacts`.
7. Run the job and inspect the downstream workspace to confirm that the files were copied into `artifacts/`.

The target `artifacts/` directory is relative to the downstream job's workspace. Jenkins creates it as part of the copy operation. The workspace is managed by Jenkins, so there is no need to create a separate server directory or change permissions on `/home/ubuntu`.

This alternative is useful when the downstream job only needs the files while it is running, or when the next build step will execute Ansible directly from the workspace. It also avoids giving a Jenkins job write access to a directory under a user's home directory.

The dedicated-directory method is more appropriate when the copied files must remain available at a predictable server path for later manual use, as in the main procedure in this project. The workspace method is more appropriate when the files are build inputs that should remain within Jenkins' normal workspace lifecycle. A workspace can be cleaned or replaced, so do not treat it as permanent storage.

The two methods are alternatives. Do not configure the same downstream job to write to both locations unless there is a specific reason to maintain both copies.

---

## Step 2 - Refactor Ansible code by importing other playbooks into `site.yml`

Before refactoring, pull the latest code from the repository's primary branch (`main` or `master`, depending on the branch in use), then create a branch named `refactor`.

![Create the refactor branch](<./images/create-branch-refactor.png>)

The DevOps philosophy emphasizes continuous improvement. Refactoring is useful when it has a clear purpose, such as reducing duplication, improving maintainability, or making future changes safer. Existing code should not be changed without a defined reason.

In the previous project, all tasks were defined in a single playbook, `common.yml`. The playbook currently contains a relatively small set of instructions for two operating-system families. As the number of tasks and target server types increases, maintaining all configuration logic in one file becomes less practical.

In that situation, the entire playbook must be reviewed to determine which tasks apply to each server or operating-system family. As the configuration grows, this review becomes time-consuming. The playbook can also become difficult to maintain if it accumulates commented-out sections and unrelated tasks. Separating configuration into focused playbooks improves readability and makes collaboration easier.

Code reuse is implemented by importing the child playbooks into the parent playbook.

1. Within the `playbooks` folder, create a new file named `site.yml`. This file will be the entry point into the infrastructure configuration. Other playbooks will be imported here. In other words, `site.yml` will become the parent playbook for the other playbooks that will be developed, including the `common.yml` playbook created previously.

2. Create a new folder in the root of the repository and name it `static-assignments`. This folder stores the child playbooks. `static-assignments` is an organizational choice rather than a special Ansible directory; another repository structure can be used where appropriate. The reason for the `static` prefix is explained later in the exercise.

3. Move the `common.yml` file into the newly created `static-assignments` folder.

![Create site.yml and move the static assignment](<./images/Screenshot 2026-09-24 120437.png>)

4. Inside `site.yml`, import the `common.yml` playbook.

```yaml
---
- ansible.builtin.import_playbook: ../static-assignments/common.yml
```

The code above uses Ansible's built-in `import_playbook` feature. It is a static import and must be declared at the top level of the playbook, not inside a play containing `hosts:`. The imported playbook contains its own play definitions, including its target hosts and tasks.

The folder structure should look like this:

```text
├── static-assignments
│   └── common.yml
├── inventory
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
└── playbooks
    └── site.yml
```

### 5. Run the `ansible-playbook` command against the `dev` environment

Because the configuration must apply tasks to the `dev` servers, where `wireshark` is already installed, an additional playbook named `common-del.yml` is created under `static-assignments`.


In that playbook, configure the removal of the `wireshark` utility.

```yaml
---
- name: Remove Wireshark from web, NFS and database servers
  hosts: webservers,nfs,db
  remote_user: ec2-user
  become: true
  tasks:
    - name: Remove Wireshark on RHEL-based servers
      ansible.builtin.dnf:
        name: wireshark
        state: absent

- name: Remove Wireshark from the load balancer
  hosts: lb
  remote_user: ubuntu
  become: true
  tasks:
    - name: Remove Wireshark on the Ubuntu load balancer
      ansible.builtin.apt:
        name: wireshark
        state: absent
        purge: true
```

The first play targets the RHEL-based web, NFS, and database servers and uses the DNF package manager. The second play targets the Ubuntu load balancer and uses APT. `become: true` elevates the task privileges, so a separate `become_user: root` setting is not required.

The `purge` option removes the package and its system-wide configuration files. The task deliberately does not run `autoremove`, because that can remove other packages that APT considers unused. It also does not run `autoclean`, which is a package-cache maintenance operation rather than a requirement for uninstalling Wireshark.

![The common-del.yml playbook](<./images/Screenshot 2026-09-24 121610.png>)

Temporarily update `site.yml` to import `common-del.yml` instead of `common.yml`.

```yaml
---
- ansible.builtin.import_playbook: ../static-assignments/common-del.yml
```

![Update site.yml to import common-del.yml](<./images/Screenshot 2026-09-24 121730.png>)

Run it against the development servers:

```bash
cd /home/ubuntu/ansible-config-mgt/

ansible-playbook -i inventory/dev.yml playbooks/site.yml
```

![Run the playbook to remove Wireshark](<./images/Screenshot 2026-09-24 131943.png>)

Ensure that Wireshark has been removed from all the targeted servers. Run the following command on each server to check:

```bash
wireshark --version
```

If the package has been removed, the command should report that the executable cannot be found. The exact message may differ between operating systems.

Check the NFS server:

![Check Wireshark on the NFS server](<./images/Screenshot 2026-09-24 131548.png>)

Check Web Server 1:

![Check Wireshark on Web Server 1](<./images/Screenshot 2026-09-24 131617.png>)

Check Web Server 2:

![Check Wireshark on Web Server 2](<./images/Screenshot 2026-09-24 131721.png>)

Check the database server:

![Check Wireshark on the database server](<./images/Screenshot 2026-09-24 131729.png>)

Check the load balancer:

![Check Wireshark on the load balancer](<./images/Screenshot 2026-09-24 13172.jpg>)

This procedure demonstrates how to use `import_playbook` and have a reusable way to install or remove packages on multiple servers with a single command.

---

## Step 3 - Configure UAT Webservers with a role `Webserver`

The development environment is already configured. This stage provisions two additional web servers for UAT. A dedicated role is used to keep web-server configuration reusable and easier to maintain, rather than placing all configuration tasks in a single playbook.

1. Launch two fresh EC2 instances using a RHEL 9 image. These instances serve as the UAT servers and are named: `Web1-UAT` and `Web2-UAT`.

![Launch the two UAT web servers](<./images/Screenshot 2026-09-24 133320.png>)

2. To create a role, create a directory called `roles/` in a location Ansible searches for roles. A common project structure is to keep `roles/` in the repository root and configure Ansible's `roles_path` to point to it.

There are two ways to create this folder structure.

Use the Ansible utility `ansible-galaxy` inside the `ansible-config-mgt/roles` directory. Create the `roles` directory first if it does not already exist:

```bash
cd /home/ubuntu/ansible-config-mgt

mkdir -p roles
cd roles
ansible-galaxy init webserver
```

**Note:** Either approach is valid. When the code is maintained in GitHub, creating and maintaining the directories and files in the repository keeps the configuration version-controlled and avoids undocumented changes on the `Jenkins-Ansible` server.

The entire folder structure should look like the following. For a manually created role, the `tests`, `files`, and `vars` directories can be omitted. When `ansible-galaxy init` is used, these directories can be retained or removed if they are not required.

![Generate the Ansible role structure](<./images/Screenshot 2026-09-24 134611.png>)

```text
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml
```

After removing unnecessary directories and files, the role structure should look like this:

```text
└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates
```


3. Restrict SSH access with EC2 security groups.

SSH access to the UAT instances is controlled at the EC2 security-group level. The application instances do not need publicly reachable SSH ports. Ansible connects to their private IP addresses from the `Jenkins-Ansible` controller, provided that the controller and UAT instances have network connectivity through the same VPC or an appropriately routed network.

Configure the security groups before testing Ansible connectivity:

- Identify the security group attached to the `Jenkins-Ansible` EC2 instance. This is the controller security group.
- Identify or create a security group for the UAT web instances.
- In the UAT web-server security group, configure an inbound rule for **SSH (TCP 22)** with the **source set to the controller security group ID**. Do not use `0.0.0.0/0` or `::/0` as the SSH source.
- Confirm that the controller security group permits outbound SSH traffic to the UAT instances. If outbound rules have been restricted, allow TCP 22 to the UAT security group or its private subnet range.
- If administrators need to SSH into the controller, restrict the controller security group's inbound SSH rule to the administrator's known public IP address or approved VPN range. Do not expose controller SSH to the entire internet.
- Remove any existing SSH inbound rules on the UAT security group that allow access from all IPv4 or IPv6 addresses.

A security-group reference is preferable to a fixed public-IP allowlist for controller-to-instance communication because the rule follows the controller security-group membership. This requires the instances to be in a network where security-group referencing is supported, such as the same VPC.

The UAT inventory must use each instance's **private IP address**. The `Jenkins-Ansible` controller must be able to route to those addresses, and the instance operating-system firewall must also permit SSH from the controller. Security groups are stateful, so return traffic for an allowed SSH connection is automatically permitted.

HTTP access is separate from SSH access. For an internal test, run `curl` from the controller against each instance's private IP address. If the application must be viewed from a browser outside the VPC, use an approved ingress path such as a load balancer or a narrowly restricted HTTP rule. The application servers do not need public SSH access for either approach.

4. Update the inventory file, `ansible-config-mgt/inventory/uat.yml`, with the private IP addresses of the two UAT web servers.

**Note:** The SSH key or forwarded SSH agent must be available to the machine and user account that execute Ansible. An SSH agent running on a local workstation does not automatically make its keys available on the remote `Jenkins-Ansible` server. When agent forwarding is used, forwarding must be enabled and the forwarded key must be visible in the remote session.

The following example shows valid YAML inventory syntax. The placeholder values represent the private IP addresses of the two UAT servers:

```yaml
---
all:
  children:
    uat-webservers:
      hosts:
        web1-uat:
          ansible_host: <Web1-UAT-Server-Private-IP-Address>
          ansible_user: ec2-user
        web2-uat:
          ansible_host: <Web2-UAT-Server-Private-IP-Address>
          ansible_user: ec2-user
```

![Update the UAT inventory](<./images/Screenshot 2026-09-24 162843.png>)

The following videos demonstrate SSH agent forwarding and connecting VS Code to the `Jenkins-Ansible` instance:

- For Windows users: [SSH agent on Windows](https://www.youtube.com/watch?v=TYyTXxVWOYA)
- For Linux users: [SSH agent on Linux](https://www.youtube.com/watch?v=EoLrCX1VVog)

Before running the playbook, test that the inventory is valid and that Ansible can connect to the UAT servers:

```bash
ansible-inventory -i inventory/uat.yml --graph

ansible -i inventory/uat.yml uat-webservers \
  -m ansible.builtin.ping
```

The SSH connection test must succeed for both hosts before playbook execution.

5. Configure Ansible's role search path. The original procedure uses `/etc/ansible/ansible.cfg`. Open that file and set `roles_path` to the location of the role directory.

When the playbook is run from the dedicated artifact directory later in this project, the copied role directory will be:

```text
/home/ubuntu/ansible-config-artifact/roles
```

Therefore, for the main directory-based artifact method, set:

```ini
[defaults]
roles_path = /home/ubuntu/ansible-config-artifact/roles
```

If `/etc/ansible/ansible.cfg` already contains a `[defaults]` section, add or update `roles_path` inside that section instead of creating a second one.

Ansible can use a configuration file from several locations, and a project-level `ansible.cfg` or the `ANSIBLE_CONFIG` environment variable can take precedence over `/etc/ansible/ansible.cfg`. Confirm which configuration file is active before relying on this setting:

```bash
ansible --version
ansible-config dump --only-changed
```

If the command reports a different configuration file, update the active configuration rather than assuming `/etc/ansible/ansible.cfg` is being used.

![Configure   Ansible roles_path](<./images/Screenshot 2026-09-24 164046.png>)

6. Add the required tasks to the webserver role. Go into the `tasks` directory and edit the `main.yml` file to perform the following tasks:

- Install and configure Apache (`httpd` service).
- Install Git so the repository can be retrieved.
- Clone the Tooling website repository from GitHub: `https://github.com/<your-name>/tooling.git`.
- Deploy the website content to `/var/www/html` on each of the two UAT web servers.
- Ensure that the `httpd` service is started and enabled.

The original procedure cloned the repository directly into `/var/www/html` and then copied a nested `html` directory into the web root. Keeping the checkout separate from the Apache document root avoids copying a directory into itself and makes the deployment paths easier to understand.

The following example assumes the repository contains the website files in a directory named `html/`. The repository URL is replaced with the URL of the intended Tooling repository.

The role task file, `tasks/main.yml`, contains the following tasks:

```yaml
---
# tasks file for webserver

- name: Install Apache
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: Install PHP packages
  ansible.builtin.dnf:
    name:
      - php
      - php-common
    state: present

- name: Install Git
  ansible.builtin.dnf:
    name: git
    state: present

- name: Clone the Tooling repository
  ansible.builtin.git:
    repo: https://github.com/<your-name>/tooling.git
    dest: /opt/tooling
    version: HEAD
    update: true
    force: false

- name: Ensure the Apache document root exists
  ansible.builtin.file:
    path: /var/www/html
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Deploy the website content to the Apache document root
  ansible.builtin.copy:
    src: /opt/tooling/html/
    dest: /var/www/html/
    remote_src: true
    owner: root
    group: root
    mode: preserve

- name: Start and enable Apache
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

The Git checkout is stored in `/opt/tooling`, while the web content is deployed from `/opt/tooling/html/` into `/var/www/html/`. The trailing slash on the source directory indicates that its contents should be copied into the destination directory.

The `force: false` setting avoids deliberately overwriting local changes in the checkout. If a local modification prevents an update, the change should be reviewed rather than automatically discarded. The `version: HEAD` setting uses the repository's default remote revision; a specific branch, tag, or commit can be configured for a controlled deployment.

The `mode: preserve` setting preserves the source file modes. Review the deployed files to ensure they are readable by the Apache process and are not writable by unrelated users. The task uses privilege escalation because the destination is under `/var/www`.

**Application runtime note:** Configure the installation of PHP or configure a PHP handler. The later browser test uses `index.php`; that URL will only execute PHP if the application requires PHP and the appropriate PHP runtime and Apache/PHP-FPM integration have also been installed and configured. Confirm the Tooling application's requirements and add the matching RHEL 9 packages and service configuration if PHP is required. Do not assume that installing `httpd` alone is sufficient to execute PHP.

![Configure the webserver role tasks](<./images/Screenshot 2026-09-24 164624.png>)

---

## Step 4 - Reference the `Webserver` role

A new assignment named `uat-webservers.yml` is created in the `static-assignments` folder. This file references the role.

```yaml
---
- name: Configure UAT webservers
  hosts: uat-webservers
  become: true
  roles:
    - webserver
```

The `become: true` setting on the UAT assignment applies privilege escalation to the role's tasks. A role's `tasks/main.yml` file contains a list of tasks only; it must not contain a play-level `hosts:` declaration or a `tasks:` wrapper.

![Create the UAT webservers assignment](<./images/Screenshot 2026-09-24 164905.png>)

The `site.yml` file remains the entry point to the Ansible configuration. The UAT assignment is therefore imported from `site.yml`.

After the Wireshark-removal test in Step 2, change `site.yml` back to the normal configuration and include both the common configuration and the UAT webserver assignment.

The correct `site.yml` structure is:

```yaml
---
- ansible.builtin.import_playbook: ../static-assignments/common.yml

- ansible.builtin.import_playbook: ../static-assignments/uat-webservers.yml
```

Each `import_playbook` statement is a top-level entry. A separate `- hosts: all` or `- hosts: uat-webservers` play is not placed immediately before an import because the imported playbooks already define their own hosts.

![Update site.yml to import the common and UAT playbooks](<./images/Screenshot 2026-09-25 155106.png>)

---

## Step 5 - Commit & Test

The changes are committed to a feature branch, submitted through a Pull Request, reviewed, and merged into the branch used by the workflow (`main` or `master`). The GitHub webhook is then verified to trigger the two Jenkins jobs in sequence. Both jobs must complete successfully, and the downstream job must copy the archived files to the `Jenkins-Ansible` server under:

```text
/home/ubuntu/ansible-config-artifact/
```

![Check the Jenkins builds](<./images/Screenshot 2026-09-24 174444.png>)

![Check the copied artifacts on the Jenkins-Ansible server](<./images/Screenshot 2026-09-25 121654.png>)

The playbook is executed against the UAT inventory after the copied artifacts and SSH connectivity have been verified.

**Note:** The `Jenkins-Ansible` server must have access to the SSH key or forwarded SSH agent used by Ansible to reach the UAT servers. The copied artifact directory must contain the inventory and role directories before execution.

For Windows users, see the [SSH agent on Windows](https://www.youtube.com/watch?v=TYyTXxVWOYA). For Linux users, see [SSH agent on Linux](https://www.youtube.com/watch?v=EoLrCX1VVog).

```bash
cd /home/ubuntu/ansible-config-artifact

# Confirm that the copied project structure is present
ls -la
ls -la inventory playbooks static-assignments roles

# Confirm the inventory can be parsed
ansible-inventory -i inventory/uat.yml --graph

# Run the site playbook
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```

The command uses paths relative to the artifact directory. It assumes the copied artifact contains `inventory/uat.yml`, `playbooks/site.yml`, `static-assignments/`, and `roles/webserver/`.

### Validate the final playbook, lint the code, and verify idempotency

The validation sequence is performed against the final files in the artifact directory. The commands below assume the active inventory is `inventory/uat.yml` and the entry-point playbook is `playbooks/site.yml`.

#### 1. Run the Ansible syntax check

```bash
ansible-playbook --syntax-check \
  -i inventory/uat.yml \
  playbooks/site.yml
```

A successful syntax check ends with a message similar to:

```text
playbook: playbooks/site.yml
```

This check validates playbook syntax and import resolution. It does not prove that SSH connectivity, privilege escalation, package installation, or application deployment will succeed.

#### 2. Run `ansible-lint` when available

Check whether `ansible-lint` is installed:

```bash
command -v ansible-lint
```

If it is available, run it against the entry point and the project components:

```bash
ansible-lint playbooks/site.yml \
  static-assignments/common.yml \
  static-assignments/uat-webservers.yml \
  roles/webserver
```

If the executable is not installed, install it in the Ansible controller's managed Python environment using the installation method approved for that environment, then repeat the lint command. Avoid installing Python tooling into a system-managed Python environment with an unrestricted `pip install`.

Review each finding and correct meaningful issues in the playbook, role, or task definitions. Do not silence findings merely to obtain a clean report. If a finding is intentionally not applicable, document the reason and use a narrowly scoped exclusion only where justified. Rerun both `ansible-lint` and the syntax check after making corrections.

#### 3. Execute the final playbook twice without changing it

After syntax validation and lint review, execute the same playbook against the same inventory twice. The playbook, inventory, role, and variables remain unchanged between the two runs.

First run:

```bash
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```

Second run, using the exact same command:

```bash
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```

The second execution is the idempotency check. For tasks that are already in the desired state, Ansible should report no additional changes. A successful second-run recap should show:

- `failed=0` for every host;
- `unreachable=0` for every host; and
- `changed=0` for every host, assuming the managed state and remote Git revision did not change between executions.

The Git task currently uses `version: HEAD`. For a repeatable test, ensure the remote repository's default branch does not change between the two runs. Pinning the task to a known commit is preferable when a fully reproducible deployment is required.

Record the actual second-run `PLAY RECAP` from the terminal in the project evidence. The actual host counts must be retained; an expected or illustrative recap is not a substitute for execution output. If any host reports `changed` on the second run, inspect the task responsible and determine whether the change is expected, caused by an external state change, or indicates a task that is not idempotent.


The `roles_path` configured in Step 3 must point to `/home/ubuntu/ansible-config-artifact/roles` when using this directory-based method. Also confirm the active Ansible configuration with `ansible --version` if Ansible cannot locate the role.

![Run the playbook to configure the UAT servers](<./images/Screenshot 2026-09-25 170138.png>)

Successful execution should leave both UAT web servers configured. Because SSH is restricted to the controller security group and the inventory uses private IP addresses, the application instances do not need public SSH exposure.

For an internal HTTP check, run the following commands from the `Jenkins-Ansible` controller. Replace the private IP placeholders with the addresses in `inventory/uat.yml`:

```bash
curl -I http://<Web1-UAT-Private-IP>/
curl -I http://<Web2-UAT-Private-IP>/
```

If the application has a PHP entry point and PHP has been installed and configured, the endpoint can also be tested from the controller:

```bash
curl -I http://<Web1-UAT-Private-IP>/index.php
curl -I http://<Web2-UAT-Private-IP>/index.php
```

To view the application from an external browser, route HTTP traffic through an approved ingress point, such as a load balancer, or use a temporary HTTP rule restricted to a trusted source IP. A public SSH rule is not required for browser testing and must not be introduced for that purpose. The security group should allow HTTP from the intended ingress source only.

**Access Web1-UAT**

![Access Web1-UAT](<./images/Screenshot 2026-09-26 150424.png>)

**Access Web2-UAT**

![Access Web2-UAT](<./images/Screenshot 2026-09-26 173611.png>)

**The resulting Ansible architecture is shown below:**

![Ansible architecture](<./images/architecture1.png>)
