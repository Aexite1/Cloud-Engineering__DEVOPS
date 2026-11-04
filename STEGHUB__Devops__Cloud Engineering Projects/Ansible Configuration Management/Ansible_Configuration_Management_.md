# Ansible Configuration Management (Automate Projects 7--10)

Projects 7 to 10 involve several manual tasks: provisioning virtual
servers, installing and configuring software, and deploying the web
application. Repeating these operations across multiple machines takes
time and makes it easier for configurations to drift.

This project introduces **Ansible Configuration Management** to automate
many of those routine tasks. It also provides practical experience
writing declarative configuration in YAML. Ansible describes the desired
state of managed hosts, then applies the required changes to bring them
into that state.

## Ansible Client as a Jump Server (Bastion Host)

A **jump server**, also known as a **bastion host**, is an intermediary
system used to access servers on a protected network. In this
architecture, the web, database, NFS and load-balancer servers are
placed in private subnets and are reached through the Ansible client in
the public subnet.

Rather than exposing every managed server directly to the internet,
administrators connect to the jump server and use it to reach internal
hosts over their private IP addresses. This reduces the number of
systems exposed to direct internet access. The security benefit depends
on restricting inbound access to the jump server and allowing SSH to the
private hosts only from trusted sources, such as the jump server's
private address or security group.

## Task

-   Install and configure an Ansible client that will also serve as a
    jump server.
-   Create a basic Ansible playbook to automate configuration tasks
    across the managed servers.

The architecture diagram shows a VPC divided into public and private
subnets. Instances in the public subnet have public IP connectivity,
while the private-subnet instances are accessed through their private IP
addresses.

![VPC architecture](<./images/architecture.png>)

## Step 1 -- Install and Configure Ansible on an EC2 Instance

### 1. Rename the Jenkins EC2 instance

In the EC2 console, update the **Name** tag of the existing Jenkins
instance to `Jenkins-Ansible`. This instance will be used to execute
Ansible playbooks.

![Rename the EC2 instance](<./images/Screenshot 2026-09-22 144118.png>)

### 2. Create the Ansible configuration repository

Create a new repository in your GitHub account and name it
`ansible-config-mgt`. The repository will hold the inventories,
playbooks and other Ansible configuration files used in this project.

![Create the GitHub repository](<./images/Screenshot 2026-09-22 144100.png>)

### 3. Install Ansible

The commands below install Ansible from the Ubuntu APT repositories.
This is the package-management method used in this implementation; it is
separate from installing Ansible with `pip`.

See the [official Ansible installation
guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
for other supported installation methods and platform-specific guidance.

Update the package index first:

``` bash
sudo apt update
```

![Update Ubuntu package lists](<./images/Screenshot 2026-09-22 144419.png>)

Install Ansible:

``` bash
sudo apt install ansible
```

![Install Ansible](<./images/Screenshot 2026-09-22 144454.png>)

Confirm that Ansible is available and check the installed version:

``` bash
ansible --version
```

![Check the Ansible version](<./images/Screenshot 2026-09-22 150044.png>)

The version reported here depends on the Ubuntu release and the
repositories configured on the instance.

### 4. Configure Jenkins to archive repository changes

Jenkins will monitor the `ansible-config-mgt` repository and archive its
contents whenever changes are pushed to the `main` branch. This reuses
the Jenkins configuration skills from Project 9.

First, configure a webhook in GitHub:

1.  Open the `ansible-config-mgt` repository.
2.  Go to **Settings → Webhooks → Add webhook**.
3.  Configure the webhook to notify Jenkins when the repository is
    updated. Use the Jenkins webhook endpoint and event settings
    required by your Jenkins GitHub integration.

![Configure the GitHub webhook](<./images/Screenshot 2026-09-22 145853.png>)

Next, create a Freestyle project in Jenkins and name it `ansible`.

![Create the Jenkins Freestyle
project](<./images/Screenshot 2026-09-22 151700.png>)

Copy the HTTPS or SSH URL of the `ansible-config-mgt` repository,
depending on the authentication method you choose.

![Copy the repository URL](<./images/Screenshot 2026-09-22 151827.png>)

In the Jenkins job configuration, select **Git** under Source Code
Management and enter the repository URL. Configure the appropriate
branch specifier for `main` and select credentials that allow Jenkins to
read the repository.

For HTTPS authentication, use a GitHub Personal Access Token (PAT)
rather than a GitHub account password. Store the token in **Jenkins
Credentials** and reference that credential from the job; do not paste
the token into the repository or commit it to source control. Give the
token only the repository access needed by this job. SSH authentication
with a dedicated deploy key is another option.

![Configure Git source control in
Jenkins](<./images/Screenshot 2026-09-22 152418.png>)

![Configure the Jenkins GitHub
webhook](<./images/Screenshot 2026-09-22 152511.png>)

Under **Post-build Actions**, configure Jenkins to archive the
repository files. Use the archive pattern `**` to include the
checked-out repository contents, following the approach used in [Project
9](https://github.com/francdomain/StegHub_DevOps-Cloud_Engineering/blob/main/Tooling-Website-Deployment-Automation-With-CI/Tooling_Website_Deploment_With_CI.md).

![Configure Jenkins artifact
archiving](<./images/Screenshot 2026-09-22 152635.png>)

### 5. Test the Jenkins job

Make a small change to `README.md` on the repository's `main` branch and
push the change. The GitHub webhook should notify Jenkins and trigger
the `ansible` job automatically.

![Edit the README file](<./images/Screenshot 2026-09-22 152836.png>)

Open the `ansible` project in Jenkins and check whether a new build has
started.

![Check the README-triggered build](<./images/Screenshot 2026-09-22 153833.png>)

Review the console output to confirm that Jenkins checked out the
repository and completed the build steps.

![Review the Jenkins console
output](<./images/Screenshot 2026-09-22 15390.jpg>)

Jenkins stores the archived files under the build's archive directory.
Replace `<build_number>` with the relevant build number:

``` bash
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

![Check the archived files from the
terminal](<./images/Screenshot 2026-09-22 154210.png>)

**Note:** Configure the job to build from the `main` branch. The webhook
may notify Jenkins about other repository events, but the job's branch
configuration should ensure that it builds the intended branch.

At this stage, the architecture includes the Jenkins-Ansible instance as
the public entry point for managing the private servers.

![Updated Ansible architecture](<./images/architecture2.png>)

**Tip:** You can associate an Elastic IP with the Jenkins-Ansible
instance to keep its public IPv4 address stable when the instance is
stopped and started. This avoids having to update the GitHub webhook
each time the instance receives a different public address.

Allocate and Associate the Elastic IP with the Jenkins-Ansible instance:

![Associate the Elastic IP](<./images/Screenshot 2026-09-22 154346.png>)

![Elastic IP associated with the
instance](<./images/Screenshot 2026-09-22 155317.png>)

If the public address changes or the webhook configuration needs
updating, update the webhook to use the instance's current address:

![Update the GitHub webhook](<./images/Screenshot 2026-09-22 154601.png>)

**Cost note:** AWS charges for public IPv4 addresses, including Elastic
IP addresses associated with running instances. Check the current
[Amazon EC2 public IPv4 address
pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Public_IPv4_Address)
for the applicable rate. Release an Elastic IP when it is no longer
required so that you do not continue incurring charges for an unused
address.

## Step 2 -- Prepare the Development Environment with Visual Studio Code

### 1. Choose a code editor

The development part of DevOps involves writing and maintaining
configuration files, scripts and application code. A suitable editor
makes these tasks easier to manage and debug.

There are many IDEs and source-code editors available, each with
different features. This project uses [Visual Studio
Code](https://code.visualstudio.com/download), a free editor that
supports a wide range of languages and extensions.

### 2. Connect Visual Studio Code to GitHub

After installing Visual Studio Code, connect it to the
`ansible-config-mgt` repository. The screenshot below shows the
repository-cloning workflow.

![Clone the repository in Visual Studio
Code](<./images/Screenshot 2026-09-22 162150.png>)

Use the repository URL:

``` text
https://github.com/Aexite1/ansible-config-mgt.git
```

### 3. Clone the repository onto the Jenkins-Ansible instance

The repository is also needed on the Jenkins-Ansible server, where the
playbooks will be executed. Clone it onto that instance using Git:

``` bash
git clone <ansible-config-mgt repo link>
```

Replace the placeholder with the URL of your own repository.

![Clone the repository onto the Ansible
server](<./images/clone-repo-to-ansible-server.png>)

## Step 3 -- Begin Ansible Development

### 1. Create a feature branch

Create a separate branch in the `ansible-config-mgt` repository for the
new configuration work. Developing on a feature branch keeps changes
isolated until they have been reviewed and are ready to merge.

Choose a branch name that describes the work. If the team uses a
project-management system such as Jira or Trello, include the related
ticket number. A naming pattern such as `feature/prj-145-lvm` identifies
both the type of work and the task it belongs to.

For this implementation, create the branch with:

``` bash
git checkout -b feature/prj-11-ansible-config
```

![Create the feature branch](<./images/Screenshot 2026-09-22 162545.png>)

### 2. Check out the feature branch locally

Fetch the latest remote branch information, then switch to the feature
branch:

``` bash
git fetch
git checkout feature/prj-11-ansible-config
```

Run these commands in the local repository clone where you intend to
develop.

### 3. Create the playbooks directory

Create a directory named `playbooks`. It will contain the Ansible
playbook files:

``` bash
mkdir playbooks
```

### 4. Create the inventory directory

Create another directory named `inventory`. This directory will hold the
inventory files used to group hosts by environment:

``` bash
mkdir inventory
``` 

![Create the inventory and playbooks
directories](<./images/Screenshot 2026-09-22 162723.png>)

![Create the inventory and playbooks
directories](<./images/Screenshot 2026-09-22 162737.png>)

### 5. Create the common playbook

Inside `playbooks`, create a file named `common.yml`. This will contain
tasks that can be applied to more than one group of servers:

``` bash
touch playbooks/common.yml
``` 
**
``` powershell
New-Item playbooks/common.yml
```

### 6. Create inventory files for each environment

Create a YAML inventory file for each environment:

-   `dev.yml` for Development
-   `staging.yml` for Staging
-   `uat.yml` for User Acceptance Testing
-   `prod.yml` for Production

Create the files with:

``` bash
touch inventory/dev.yml inventory/staging.yml inventory/uat.yml inventory/prod.yml
```

These files use **YAML inventory syntax**. They are not INI-style
inventories; the file extension and the YAML structure used later in
this project should remain consistent.

![Create the inventory and playbook
files](<./images/Screenshot 2026-09-22 163221.png>)

![Inventory and playbook directory
structure](<./images/Screenshot 2026-09-22 163300.png>)

## Step 4 -- Configure an Ansible Inventory

An Ansible inventory identifies the managed hosts and organizes them
into groups. Playbooks use these groups to determine which machines
should receive particular tasks. Since this project runs Linux commands
and configuration tasks on remote servers, the inventory provides a
central place to define the target hosts and their connection details.

The first inventory to configure is `inventory/dev.yml`. Add the private
IP addresses of the servers in your own environment, replacing the
placeholders in the example below.

### Prepare SSH authentication

Ansible connects to Linux targets over SSH, which uses TCP port 22 by
default. The Jenkins-Ansible instance therefore needs a way to
authenticate to each managed server.

This walkthrough uses an SSH agent to make a locally loaded key
available to the SSH session. Agent forwarding is convenient, but it
should only be enabled when necessary and when the intermediate host is
trusted. A forwarded agent does not copy the private key to the remote
machine, but a compromised remote host may be able to request
authentication operations from the forwarded agent while the connection
is active. For a persistent automation setup, a dedicated key with
restricted access is preferable to forwarding a personal key.

For guidance on connecting to the Jenkins-Ansible instance and using an
SSH agent, see:

-   [SSH agent on Windows](https://www.youtube.com/watch?v=OplGrY74qog)
-   [SSH agent on Linux](https://www.youtube.com/watch?v=RRRQLgAfcJw)

Start the SSH agent in the terminal session:

``` bash
eval "$(ssh-agent -s)"
```

This starts the agent and prints the environment variables that allow
SSH tools in the current shell to communicate with it.

Add the private key that should be used for the connection. Replace the
placeholder with the actual path to the key:

``` bash
ssh-add <path-to-private-key>
```

![Add the private key to the SSH agent](<./images/Screenshot 2026-09-23 110008.png>)

Confirm that the key has been loaded:

``` bash
ssh-add -l
```

The command should list the identities currently held by the agent.

![Verify that the private key is
loaded](<./images/Screenshot 2026-09-23 110030.png>)

Connect to the Jenkins-Ansible instance with agent forwarding enabled:

``` bash
ssh -A ubuntu@public-ip
```

Replace `public-ip` with the instance's public IP address or Elastic IP.
The `-A` option forwards access to the local SSH agent for the duration
of the SSH session. Use it only if the remote host is trusted and
forwarding is required for the next SSH connection.

![Connect to the Ansible server using agent
forwarding](<./images/Screenshot 2026-09-23 110113.png>)

![Confirm SSH access](<./images/Screenshot 2026-09-23 110218.png>)

For another walkthrough of the SSH and Visual Studio Code connection
process, see [Windows](https://www.youtube.com/watch?v=OplGrY74qog) or
[Linux](https://www.youtube.com/watch?v=RRRQLgAfcJw).

The SSH usernames depend on the operating system images used in this
infrastructure:

-   The load balancer uses `ubuntu`.
-   The RHEL-based servers use `ec2-user`.
-   The database server in this inventory also uses `ubuntu`, as shown
    in the configuration below.

### Define the development hosts

Update `inventory/dev.yml` with the following YAML inventory. Replace
each placeholder with the corresponding **private IP address**:

``` yaml
all:
  children:
    nfs:
      hosts:
        <NFS-Server-Private-IP-Address>:
          ansible_ssh_user: ec2-user
    webservers:
      hosts:
        <Web-Server1-Private-IP-Address>:
          ansible_ssh_user: ec2-user
        <Web-Server2-Private-IP-Address>:
          ansible_ssh_user: ec2-user
    db:
      hosts:
        <Database-Private-IP-Address>:
          ansible_ssh_user: ubuntu
    lb:
      hosts:
        <Load-Balancer-Private-IP-Address>:
          ansible_ssh_user: ubuntu
```

![Development inventory configuration](<./images/Screenshot 2026-09-23 115650.png>)

The inventory groups the machines into `nfs`, `webservers`, `db` and
`lb`. The `ansible_ssh_user` variable specifies the remote login account
for each host. Ansible still needs a valid SSH authentication method for
those accounts; defining the username and private IP address alone does
not provide credentials.

## Step 5 -- Create a Common Playbook

The inventory identifies the servers, while the playbook defines what
Ansible should do on them. The `common.yml` playbook will contain
repeatable tasks that can be applied to the relevant groups of machines.

Open `playbooks/common.yml` and add the following configuration:

``` yaml
---
- name: Update web and NFS servers
  hosts: webservers, nfs
  remote_user: ec2-user
  become: true
  become_user: root
  tasks:
    - name: Ensure wireshark is at the latest version
      dnf:
        name: wireshark
        state: latest

- name: Update LB and DB servers
  hosts: lb, db
  remote_user: ubuntu
  become: true
  become_user: root
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: Ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

![Common Ansible playbook in Visual Studio
Code](<./images/Screenshot 2026-09-23 110946.png>)

The playbook contains two plays because the target servers use different
Linux distributions and package managers.

The first play targets the RHEL-based web and NFS servers. It connects
as `ec2-user`, elevates privileges to root with Ansible's
privilege-escalation mechanism, and uses the `dnf` package module to
ensure that Wireshark is installed at the latest version available from
the configured repositories. `dnf` is the appropriate Ansible package
module for DNF-based systems such as RHEL 9.

The second play targets the Ubuntu load balancer and database servers.
It connects as `ubuntu`, escalates privileges, refreshes the APT package
index, and then ensures that the Wireshark package is at the latest
version available through APT.

Wireshark is retained here because it is the package used in the
original project exercise. On a headless server, its graphical interface
may not be useful; this playbook demonstrates package management rather
than configuring a graphical packet-analysis workstation. The package's
availability can also depend on the enabled repositories and the
operating-system release.

The `state: latest` setting allows Ansible to install the package if it
is absent and upgrade it when a newer repository version is available.
This is useful for demonstrating state management, but it also means a
later playbook run may upgrade the package. For environments that
require tightly controlled versions, pinning a package version may be
more appropriate.

The playbook can be extended with other configuration tasks, for
example:

-   Creating a directory and a file inside it.
-   Changing the timezone across the servers.
-   Executing a shell script.

To explore playbook structure and task execution further, see the [Red
Hat Ansible Playbook video](https://www.youtube.com/watch?v=ZAdJ7CdN7DY)
and the Red Hat article [What is an Ansible
Playbook?](https://www.redhat.com/en/topics/automation/what-is-an-ansible-playbook).

## Step 6 -- Push the Latest Code to GitHub

The inventory and playbook files now exist in the local repository. The
next step is to push the changes to GitHub so that they can be reviewed
and merged into the `main` branch.

In a team environment, changes are commonly reviewed before they are
merged into the shared branch. This practice is sometimes called the
**four-eyes principle**: another person reviews the work before it
becomes part of the main codebase.

The feature branch created earlier provides an isolated place for this
work. Once the changes are pushed, a Pull Request (PR) can be opened so
that another developer can review them.

### 1. Commit and push the changes

Check the working tree, stage the files you want to include, create a
commit and push the feature branch to GitHub:

``` bash
git status

git add <selected files>

git commit -m "commit message"

git push origin feature/prj-11-ansible-config
```

Replace `<selected files>` with the paths you intend to commit and use a
commit message that describes the change.

![Add, commit and push the feature
branch](<./images/Screenshot 2026-09-23 120445.png>)

### 2. Open a Pull Request

On GitHub, open a Pull Request from `feature/prj-11-ansible-config` into
`main`.

![Create the Pull Request](<./images/Screenshot 2026-09-23 120620.png>)

### 3. Review the proposed changes

Review the changes as another developer would. Check the playbook,
inventory and other files for mistakes before approving the Pull
Request.

![Review the Pull Request](<./images/Screenshot 2026-09-23 120620.png>)

### 4. Merge the Pull Request

Once the review is complete and the changes are approved, merge the
feature branch into `main`.

![Confirm that the branch has no merge
conflicts](<./images/Screenshot 2026-09-23 122743.png>)


### 5. Update the local main branch

Return to the terminal, switch from the feature branch to `main`, and
pull the latest changes from GitHub:

``` bash
git checkout main
git pull origin main
```

![Pull the latest changes from main](<./images/Screenshot 2026-09-23 132751.png>)

After the changes reach `main`, Jenkins should detect the update through
the configured webhook and run the `ansible` job. The job then archives
the repository files as build artifacts.

![Jenkins build after the merge](<./images/Screenshot 2026-09-23 133216.png>)

Review the build's console output:

![Jenkins console output](<./images/j-console-output.png>)

The archived files can be found in the following directory. Replace
`<build_number>` with the build number generated by Jenkins:

``` bash
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

![Inspect the archived build files](<./images/Screenshot 2026-09-23 133413.png>)

## Step 7 -- Run the First Ansible Test

With the inventory and playbook in place, the next step is to execute
the playbook from the Jenkins-Ansible server and verify the result on
the managed hosts.

### 1. Install the Visual Studio Code Remote Development extensions

Install the **Remote Development** extension pack or the **Remote -
SSH** extension in Visual Studio Code. These extensions allow the editor
to connect to a remote machine over SSH and work with files on that
machine.

![Install the Remote Development
extension](<./images/Screenshot 2026-09-23 133846.png>)

### 2. Configure the SSH host

Add the Jenkins-Ansible instance to the SSH configuration used by Visual
Studio Code, then connect to it through the Remote - SSH extension.

![Configure the remote SSH host](<./images/Screenshot 2026-09-23 134817.png>)

A new Visual Studio Code window should open for the remote connection.
The status indicator should show `SSH: jenkins-ansible`, confirming that
the editor is connected to the remote server rather than working only
with local files.

![Confirm the remote connection and open the
inventory](<./images/Screenshot 2026-09-23 135351.png>)

![Open the playbooks directory on the remote
server](<./images/remote-connect-playbooks.png>)

### 3. Execute the playbook

From the repository directory on the Jenkins-Ansible instance, run:

``` bash
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```

The `-i` option tells Ansible which inventory file to use. The final
argument specifies the playbook to execute. Ansible reads the host
groups from `inventory/dev.yml`, matches them with the plays in
`common.yml`, connects to the selected servers and runs the defined
tasks.

![Run the playbook and install
Wireshark](<./images/Screenshot 2026-09-23 171415.png>)

### 4. Verify the package on each server

Connect to each managed server and check whether the Wireshark
executable is available:

``` bash
which wireshark
```

Alternatively, check the installed version:

``` bash
wireshark --version
```


Check Web Server 1:

![Verify Wireshark on Web Server 1](<./images/Screenshot 2026-09-23 171748.png>)

Check Web Server 2:

![Verify Wireshark on Web Server 2](<./images/Screenshot 2026-09-23 171753.png>)

Check the NFS server:

![Verify Wireshark on the NFS server](<./images/Screenshot 2026-09-23 171942.png>)

Check the database server:

![Verify Wireshark on the database server](<./images/Screenshot 2026-09-23 172201.png>)

Check the load balancer:

![Verify Wireshark on the load balancer](<./images/Screenshot 2026-09-23 172429.png>)

Once the playbook has completed and the package has been verified, the
architecture includes the Ansible-managed configuration workflow.

![Final Ansible architecture](<./images/architecture3.png>)

## Optional -- Repeat the Workflow

Add another task to the Ansible playbook and repeat the development and
delivery process:

1.  Check out the feature branch and make the change.
2.  Commit and push the updated code.
3.  Open a Pull Request and have the changes reviewed.
4.  Merge the approved changes into `main`.
5.  Allow Jenkins to archive the updated repository.
6.  Run the Ansible playbook to apply the new configuration.

Repeating this cycle provides more practice with Git collaboration,
Jenkins automation and Ansible playbook execution. The same approach can
be used to manage routine configuration tasks across a larger group of
servers.

This project has introduced a workflow in which configuration is stored
as code, changes are reviewed through GitHub, Jenkins archives the
repository, and Ansible applies the defined tasks to the managed
infrastructure.

------------------------------------------------------------------------

## Technical notes and limitations

-   The playbook assumes that the RHEL-based hosts use DNF and that the
    Ubuntu hosts use APT. The package must be available in the
    repositories configured on each host.
-   The inventory uses private IP addresses and assumes that the
    Jenkins-Ansible instance can reach the managed hosts over the
    network.
-   SSH connectivity depends on valid usernames, key authentication and
    security-group or firewall rules. The inventory alone does not
    configure those network permissions.
-   SSH agent forwarding is shown to match the original workflow. For a
    long-running automation environment, a dedicated and restricted
    automation key is preferable.
-   The Jenkins job must be configured to build the `main` branch, and
    the GitHub webhook must point to the correct Jenkins endpoint. The
    exact webhook event settings depend on the Jenkins plugins and job
    configuration.
-   The instructions retain Wireshark as the package being installed.
    Its graphical interface may not be usable on headless servers; the
    exercise demonstrates package installation and configuration
    management.
-   AWS public IPv4 address charges and Elastic IP pricing can change.
    Confirm the current rate on the AWS pricing page before estimating
    project costs.
