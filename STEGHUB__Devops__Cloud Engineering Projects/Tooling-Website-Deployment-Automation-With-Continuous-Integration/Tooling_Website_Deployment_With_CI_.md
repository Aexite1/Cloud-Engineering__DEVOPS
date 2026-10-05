# Tooling Website Deployment Automation with Jenkins CI

This project introduces Jenkins as the automation server for adding Continuous Integration to the existing Tooling Website environment. The goal is to automate the movement of updated application files from GitHub through Jenkins and onto the NFS server.

Continuous Integration is a workflow where code changes are processed automatically as they are committed. In this setup, Jenkins receives change notifications from GitHub, retrieves the updated source files, creates build artifacts, and prepares them for deployment.

## Project Objective

Extend the existing architecture by adding a ```Jenkins``` server. The Jenkins job will retrieve source-code changes from ```GitHub``` and automatically transfer the resulting files to the ```NFS``` server.

The updated architecture is shown below.

![diagram](<./images/architecture.png>)


# Step 1 - Provision and Install Jenkins

## 1. Launch the Jenkins EC2 instance

![](<./images/Screenshot 2026-09-17 153348.png>)
![](<./images/Screenshot 2026-09-17 153423.png>)


## 2. Prepare Java for Jenkins

__Connect to the Jenkins instance__

```bash
ssh -i "ec2-key" ubuntu@13.49.226.68
```

__Update the operating system packages__

```bash
sudo apt update && sudo apt upgrade
```
![](<./images/Screenshot 2026-09-17 153646.png>)

__Add the Jenkins signing key__

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```
![](<./images/Screenshot 2026-09-17 154317.png>)

__Register the Jenkins package repository__

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
![](<./images/Screenshot 2026-09-17 154549.png>)

__Install the Java runtime__

Jenkins runs on Java, so a compatible runtime must be available before the Jenkins package is installed. The commands below install the Java runtime used in this setup.

```bash
sudo apt install fontconfig openjdk-21-jre
```
![](<./images/Screenshot 2026-09-17 154654.png>)


## 3. Install and start Jenkins

__Refresh the package index__

```bash
sudo apt-get update
```
![](<./images/Screenshot 2026-09-17 154549.png>)

__Install the Jenkins package__

```bash
sudo apt-get install jenkins -y
```
__Enable and verify the Jenkins service__

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
![](<./images/Screenshot 2026-09-17 160408.png>)

## 4. Allow access to Jenkins on TCP port 8080

![](<./images/Screenshot 2026-09-17 160700.png>)

## 5. Complete the initial Jenkins configuration

Open the Jenkins web interface using ```http://<Jenkins-Server-Public-IP-Address>:8080```.
Jenkins will request the initial administrator password during the first launch.
Read the password directly from the Jenkins host.

```bash
http://13.49.226.68:8080
```

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
![](<./images/Screenshot 2026-09-17 160928.png>)

When Jenkins asks which plugins should be installed, select the suggested plugin set.

![](<./images/Screenshot 2026-09-17 161006.png>)
![](<./images/Screenshot 2026-09-17 161023.png>)

After the plugin installation finishes, create the administrator account and confirm the Jenkins server URL.

![](<./images/Screenshot 2026-09-17 161306.png>)

The Jenkins installation and initial setup are now complete.

![](<./images/Screenshot 2026-09-17 161319.png>)


# Step 2 - Connect Jenkins to GitHub with Webhooks

Next, create a Jenkins job that receives notifications from GitHub through a webhook. When a change is pushed, Jenkins will retrieve the repository contents and store the resulting build files locally.

## 1. Add a webhook to the GitHub repository

Open the GitHub repository and navigate to

Settings > Webhooks > Add webhook.

## 2. Create the Jenkins Freestyle project

![](<./images/Screenshot 2026-09-17 161447.png>)

__Add the GitHub repository URL__

```bash
https://github.com/francdomain/tooling.git
```
![](<./images/cp-github-repo.png>)

In the Freestyle project configuration, select Git as the source-code management option. Enter the Tooling repository URL and configure the credentials required for Jenkins to access it.

![](<./images/Screenshot 2026-09-17 162543.png>)

Save the project configuration and run the first build manually with ``Build Now``. If the repository settings and credentials are correct, Jenkins should complete the build successfully and display it in the build history.
Open the build entry and inspect its Console Output to confirm that the process completed successfully.

![](<./images/Screenshot 2026-09-17 174714.png>)

At this stage the build is manual and does not yet react to repository changes. The next configuration adds the webhook trigger and preserves the build output as artifacts.

## 3. Configure automatic triggering and artifact archiving

__Enable the GitHub webhook trigger and configure ``Post-build Actions`` to archive the build files. Files produced by a build are stored as artifacts.__

![](<./images/Screenshot 2026-09-17 174848.png>)

Make a small change to a file in the GitHub repository, such as ``README.md``, and push the update to the main branch.

![](<./images/Screenshot 2026-09-17 180043.png>)

The webhook should cause Jenkins to start a new build automatically. The generated artifacts are then retained on the Jenkins server.

![](<./images/Screenshot 2026-09-17 180232.png>)
![](<./images/Screenshot 2026-09-17 174932.png>)

The Jenkins job is now event-driven: a push to GitHub causes the webhook to notify Jenkins and start the build. Other trigger methods are also available, including ``trigger one job (downstreadm) from another (upstream)`` and ``poll GitHub periodically``.

By default, Jenkins keeps the archived artifacts on the Jenkins host.

```bash
ls /var/lib/jenkins/jobs/steghubtooling_github/builds/<build_number>/archive/
```


# Step 3 - Deploy Jenkins Artifacts to the NFS Server over SSH

With the build artifacts available on Jenkins, the deployment stage is to transfer them to the NFS server's ``/mnt/apps`` directory.

Jenkins can be extended through plugins. For this deployment step, use the ``Publish Over SSH`` plugin to transfer the build artifacts to the NFS server.

### 1. Install the Publish Over SSH plugin

From the Jenkins dashboard, open Manage Jenkins > Manage Plugins > Available. Search for ``Publish Over SSH`` and install the plugin.

![](<./images/Screenshot 2026-09-17 181301.png>)
![](<./images/Screenshot 2026-09-17 181334.png>)


### 2. Configure SSH access to the NFS server

From the Jenkins dashboard, open ``Manage Jenkins > Configure System``.

Locate the Publish over SSH configuration section and provide the connection details for the NFS server:

- Add the ``private key`` used to establish the SSH connection to the NFS server.

- Assign a descriptive name for the SSH server entry.

- Set the hostname to the ``private IP address`` of the ``NFS`` server.

- Use ``ec2-user`` because the NFS host is running on an EC2 instance with RHEL 10.

- Set the remote directory to ``/mnt/apps``, which is the shared location used by the web servers to obtain application files.

![](<./images/Screenshot 2026-09-17 191729.png>)

Test the SSH configuration and confirm that the connection reports Success. TCP port 22 must be reachable on the NFS server for this connection to work.

![](<./images/c:\Users\aexit\Documents\STEGHUB__Devops__Cloud Engineering Projects\cI\images\Screenshot 2026-09-17 191831.png.png>)


Save the global configuration, return to the Jenkins job configuration, and add the ``Send build artifact over ssh`` post-build action.

Configure the action to send the build output to the previously defined remote directory. To include all files and directories, use ``**``. For more selective transfers, the Ant pattern syntax can be used.

![](<./images/Screenshot 2026-09-17 192117s.png>)

Save the job configuration, make another change to ``README.md`` in the GitHub Tooling repository, and push the update.

![](<./images/Screenshot 2026-09-17 192348.png>)

The previously added line in ``README.md`` has been removed in the new revision.

The GitHub webhook should trigger another Jenkins build.


If the build reports a permission error, the NFS target directory may not allow ``ec2-user`` to write the transferred files. Check the ownership and permissions of the target path on the NFS server.

```bash
sudo chown -R ec2-user:ec2-user /mnt/apps
sudo chmod -R 755 /mnt/apps
```

__Run the Jenkins build again from the web interface__

After the webhook starts the job, the Console Output should report a successful file transfer, similar to:

```bash
SSH: Transferred 24 file(s)
Finished: SUCCESS
```

![](<./images/Screenshot 2026-09-18 003311.png>)


__Verify the deployed files on the NFS server__

```bash
ls /mnt/apps

or

ls -l /mnt/apps
```
![](<./images/Screenshot 2026-09-18 003420.png>)

```bash
cat /mnt/apps/README.md
```
![](<./images/Screenshot 2026-09-18 003505.png>)


If the changes pushed to GitHub are visible under ``/mnt/apps`` on the NFS server, the automated deployment flow is working as intended.

