# Ansible Refactoring & Static Assignments (Imports and Roles)
This project involves the refactoring of Ansible playbooks and the use of static assignments, imports, and roles. It demonstrates how to effectively structure an Ansible project for better maintainability, reuse, and scalability.
![](/images2/Screenshot%20(695).png)

Connect to your Jenkins-Ansible server terminal by ssh on your terminal

create a directory to store Jenkins artifacts:

```bash
sudo mkdir /home/ubuntu/ansible-config-artifact
```
Change the permissions to ensure Jenkins can write to it:

```bash
sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact
```
![](/images2/Screenshot%20(676).png)

Go to the Jenkins dashboard.login on port 8080 in our case we will login with
http://51.21.47.126:8080/

![](/images2/Screenshot%20(677).png)

Now go navigate to manage jenkins-->plugins-->available plugins

Search for the Copy Artifact plugin, then install it without restarting Jenkins.

![](/images2/Screenshot%20(678).png)
![](/images2/Screenshot%20(679).png)

Create a New Freestyle Project:

In Jenkins, create a new Freestyle Project and name it save_artifacts.

![](/images2/Screenshot%20(680).png)


Set it to be triggered by the completion of your existing Ansible job (configure this under Build Triggers and choose "Build after other projects are built" and type in "Ansible" as the project to watch).

![](/images2/Screenshot%20(681).png)


### Configure Artifact Copying:

In the Build step section of save_artifacts, add a Copy artifacts from another project step.
Set Source Project to your Ansible job(named "ansible").
Set the Target Directory to /home/ubuntu/ansible-config-artifact


![](/images2/Screenshot%20(682).png)


Test Your Setup:

Make a small change (like editing README.MD) in your Ansible repository’s main branch and commit it.
Check if both Jenkins jobs (ansible and save_artifacts) run in sequence.
If both Jenkins jobs have completed one after another - you shall see your files inside `/home/ubuntu/ansible-config-artifact` directory and it will be updated with every commit to your master branch.
Now your Jenkins pipeline is more neat and clear


If you see an error like this
```bash
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/save_artifacts
FATAL: /home/ubuntu/ansible-config-artifact
java.nio.file.AccessDeniedException: /home/ubuntu/ansible-config-artifact
	at java.base/sun.nio.fs.UnixException.translateToIOException(UnixException.java:90)
	at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:106)
	at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:111)
	at java.base/sun.nio.fs.UnixFileSystemProvider.createDirectory(UnixFileSystemProvider.java:438)
	at java.base/java.nio.file.Files.createDirectory(Files.java:699)
	at java.base/java.nio.file.Files.createAndCheckIsDirectory(Files.java:807)
	at java.base/java.nio.file.Files.createDirectories(Files.java:793)
	at hudson.FilePath.mkdirs(FilePath.java:3753)
	at hudson.FilePath$Mkdirs.invoke(FilePath.java:1419)
	at hudson.FilePath$Mkdirs.invoke(FilePath.java:1414)
	at hudson.FilePath.act(FilePath.java:1235)
	at hudson.FilePath.act(FilePath.java:1218)
	at hudson.FilePath.mkdirs(FilePath.java:1409)
	at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.copy(CopyArtifact.java:715)
	at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.perform(CopyArtifact.java:679)
	at PluginClassLoader for copyartifact//hudson.plugins.copyartifact.CopyArtifact.perform(CopyArtifact.java:563)
	at jenkins.tasks.SimpleBuildStep.perform(SimpleBuildStep.java:123)
	at hudson.tasks.BuildStepCompatibilityLayer.perform(BuildStepCompatibilityLayer.java:80)
	at hudson.tasks.BuildStepMonitor$1.perform(BuildStepMonitor.java:20)
	at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuild.java:818)
	at hudson.model.Build$BuildExecution.build(Build.java:199)
	at hudson.model.Build$BuildExecution.doRun(Build.java:164)
	at hudson.model.AbstractBuild$AbstractBuildExecution.run(AbstractBuild.java:526)
	at hudson.model.Run.execute(Run.java:1894)
	at hudson.model.FreeStyleBuild.run(FreeStyleBuild.java:44)
	at hudson.model.ResourceController.execute(ResourceController.java:101)
	at hudson.model.Executor.run(Executor.java:446)
Finished: FAILURE
```
![](/images2/Screenshot%20(683).png)

The error indicates that Jenkins doesn't have the necessary permissions to write to the directory /home/ubuntu/ansible-config-artifact. Specifically, it's an AccessDeniedException, meaning Jenkins is being blocked from creating or accessing the directory.

To resolve this, follow these steps:
Give Jenkins Access to the Directory
Change the ownership of the /home/ubuntu/ansible-config-artifact directory to the Jenkins user:

```bash
sudo chown -R jenkins:jenkins /home/ubuntu/ansible-config-artifact
```
This command will make the Jenkins user the owner of the directory, allowing it to write files there.

Then, modify the permissions to allow Jenkins to read and write to the directory:

```bash
sudo chmod -R 755 /home/ubuntu/ansible-config-artifact
```
This command sets read, write, and execute permissions for the owner (Jenkins) and read/execute permissions for others.

Verify Permissions
Run the following command to check the permissions of the directory and ensure that Jenkins has access:
```bash
ls -l /home/ubuntu/ansible-config-artifact
```
You should see that the directory is owned by jenkins:jenkins and has the appropriate permissions.

Rebuild the Jenkins Job
Once the permissions have been fixed, go back to Jenkins and rebuild the save_artifacts job.
The job should now be able to copy artifacts to the /home/ubuntu/ansible-config-artifact directory without permission errors.

![](/images2/Screenshot%20(684).png)
![](/images2/Screenshot%20(685).png)

### Refactor Ansible Code with Imports and Roles
Set Up a New Branch for Refactoring:

Pull the latest changes from the main branch of your Ansible repository.

```bash
git checkout -b refactor
```
`DevOps` philosophy implies constant iterative improvement for better efficiency - refactoring is one of the techniques that can be used, but you always have an answer to question "why?". Why do we need to change something if it works well?

In your playbooks directory, create a new file called site.yml.
This file will serve as the main entry point for all your playbooks, so you can centralize your configuration

Organize Playbooks with static-assignments:
Create a folder named static-assignments at the root of your Ansible repository.

Move the existing common.yml file into static-assignments.

![](/images2/Screenshot%20(687).png)

### Set Up Imports in site.yml:
Open site.yml and add the following content to import common.yml:
```yaml
---
- hosts: all
- import_playbook: ../static-assignments/common.yml
```
![](/images2/Screenshot%20(686).png)

### Verify Folder Structure:

Ensure your folder structure looks like this:
```arduino

├── static-assignments
│   └── common.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
```    

Run the Playbook:
Run the playbook for your dev environment to confirm it works:
```bash
ansible-playbook -i inventory/dev.ini playbooks/site.yml
  ```
### Add a New Playbook to Delete Wireshark
Create common-del.yml:
In the static-assignments folder, create a new playbook named common-del.yml.

Add tasks to delete Wireshark from different server types
```bash
---
- name: Remove Wireshark from web, nfs, and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: Delete Wireshark on RHEL/CentOS servers
      yum:
        name: wireshark
        state: removed
      when: ansible_facts['pkg_mgr'] == "yum"

- name: Remove Wireshark from LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Delete Wireshark on Ubuntu/Debian servers
      apt:
        name: wireshark
        state: absent
        purge: yes
      when: ansible_facts['pkg_mgr'] == "apt"
```
![](/images2/Screenshot%20(688).png)

Update site.yml to Include common-del.yml:

Replace the import_playbook line in site.yml with:
```yaml
---
- hosts: all
- import_playbook: ../static-assignments/common-del.yml
```

### Run the Updated Playbook:
Run the playbook again for your dev environment:
```bash
ansible-playbook -i inventory/dev.ini playbooks/site.yml
```
Now you have learned how to use import_playbooks module and you have a ready solution to install/delete packages on multiple servers with just one command.

### Verify Wireshark Removal:

Check each server to confirm Wireshark has been removed by running:
```bash
wireshark --version
```

## Configure UAT Webservers with a role `Webserver`

We have our nice and clean dev environment, so let us put it aside and configure 2 new Web Servers as uat. We could write tasks to configure Web Servers in the same playbook, but it would be too messy, instead, we will use a dedicated role to make our configuration reusable.

1. Launch 2 fresh EC2 instances using RHEL 9 image, we will use them as our uat servers, so give them names accordingly - `Web1-UAT` and `Web2-UAT`.

2. To create a role, you must create a directory called `roles/`, relative to the playbook file or in `/etc/ansible/` directory.

There are two ways how you can create this folder structure:

Use an Ansible utility called ansible-galaxy inside `ansible-config-mgt/roles` directory (you need to create `roles` directory upfront)

```bash
mkdir roles
cd roles
ansible-galaxy init webserver
```
![](/images2/Screenshot%20(689).png)

Remove unnecessary directories and files:
```bash
cd webserver
rm -rf files vars templates tests
```
![](/images2/Screenshot%20(690).png)

__Note__: You can choose either way, but since you store all your codes in GitHub, it is recommended to create folders and files there rather than locally on `Jenkins-Ansible` server.

The entire folder structure should look like below, but if you create it manually - you can skip creating `tests`, `files`, and `vars` or remove them if you used `ansible-galaxy`

```css
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
After removing unnecessary directories and files, the roles structure should look like this

```css
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

### Configure the Inventory File
Open the UAT inventory file:
```bash
sudo nano inventory/uat.ini
```

Add the private IPs of your UAT servers:
```yaml
[uat-webservers]
<Web1-UAT-Private-IP> ansible_ssh_user=ec2-user
<Web2-UAT-Private-IP> ansible_ssh_user=ec2-user
```

### Update ansible.cfg
Open the Ansible configuration file:
```bash
sudo nano /etc/ansible/ansible.cfg
```

if you encounter an error that the file does not exist

Create a Global Configuration File
Create and Edit ansible.cfg:
```bash
sudo nano /etc/ansible/ansible.cfg
```
Update the roles_path with the code below:
```ini
roles_path = /home/ubuntu/ansible-config-mgt/roles
```

![](/images2/Screenshot%20(691).png)


### Write the webserver Role Logic
It is time to start adding some logic to the webserver role. Go into `tasks` directory, and within the `main.yml` file, start writing configuration tasks to do the following:

- Install and configure Apache (httpd service)
- Clone Tooling website from GitHub https://github.com/<your-name>/tooling.git.
- Ensure the tooling website code is deployed to /var/www/html on each of 2 UAT Web servers.
- Make sure httpd service is started

Your main.yml consist of following tasks:

```yaml
# tasks file for webserver
---
- name: install apache
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.git:
    repo: https://github.com/francdomain/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  remote_user: ec2-user
  become: true
  become_user: root
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```
![](/images2/Screenshot%20(692).png)


### Referencing the 'Webserver' Role

 Navigate to the Correct Directory
Ensure you’re in the ansible-config-mgt directory where all your configuration files and roles are stored. Use this command to confirm your location:

```bash
Copy code
cd ~/ansible-config-mgt
```
Run the ls command to verify that the directory contains folders like inventory, roles, and playbooks.

### Create uat-webservers.yml
Navigate to the static-assignments folder:

```bash
cd ~/ansible-config-mgt/static-assignments
```
Create a new file named uat-webservers.yml:
```bash
nano uat-webservers.yml
```
Add the following content:
```yaml
---
- name: Configure UAT Webservers
  hosts: uat-webservers
  roles:
    - webserver
```
![](/images2/Screenshot%20(693).png)    

### Reference uat-webservers.yml in site.yml
Navigate to the playbooks directory:
```bash
cd ~/ansible-config-mgt/playbooks
```
Open the site.yml file:
```bash
sudo nano site.yml
```

Modify it to include a reference to uat-webservers.yml. Ensure your file looks like this:

```yaml
---
- hosts: all
  import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
  import_playbook: ../static-assignments/uat-webservers.yml
  ```
  ![](/images2/Screenshot%20(694).png)
Save and exit the file.

### Commit Your Changes
Now, let’s commit the changes to Git and push them to the repository.

Navigate to the root of your project:

```bash
cd ~/ansible-config-mgt
```

```bash
git add .
```
Commit the changes with a message:
```bash
Copy code
git commit -m "Added uat-webservers role and updated site.yml"
```
Push the changes to your repository:

```bash
git push origin refactor
```
Create a Pull Request (PR) on your GitHub repository hosting platform  to merge these changes into the main branch.

### Verify Jenkins Webhook
Once the PR is merged, ensure that the webhook triggers the Jenkins pipeline jobs:

Check Jenkins to see if two jobs ran:

The first job: Builds the configuration.
The second job: Deploys the files to /home/ubuntu/ansible-config-mgt on the Jenkins-Ansible server.

Confirm the files are updated on the Jenkins-Ansible server by SSHing into it:
Then change the branch to main by running git checkout main and after that run git pull origin main to update

Then 
cd /home/ubuntu/ansible-config-mgt
ls
Verify the updated uat-webservers.yml and site.yml files are present

### Run the playbook with the UAT inventory file:
```bash
ansible-playbook -i /inventory/uat.ini playbooks/site.yml
```

### Verify the Configuration
SSH into one of the UAT servers:
```bash
ssh -i <path-to-your-private-key> ec2-user@<Web1-UAT-Private-IP>
```
Check if Apache is running:
```bash
sudo systemctl status httpd
```

Verify the website files:
```bash
ls -l /var/www/html
```

### Verify Web Servers
After the playbook runs successfully, note the public IP or DNS of your UAT web servers.

Access the servers from your browser to verify:

```css

http://<Web1-UAT-Server-Public-IP-or-Public-DNS-Name>/index.php
```
or

```css
http://<Web2-UAT-Server-Public-IP-or-Public-DNS-Name>/index.php
```


### Conclusion
We refactored our ansible configuration to deploy our web application using ansible roles. We also used Ansible imports and roles to modularise our ansible configuration.