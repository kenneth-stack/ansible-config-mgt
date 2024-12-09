# Ansible Configuration management for the 3-tier architecture.
![Architecture diagram for Ansible configuration mgt](/images/Screenshot%20(664).png)

We will be automating the management of our web application with infrastructure as code (IAC) using Ansible.

Ansible helps to speed up deployment times by automating the routine tasks of setting up servers, databases etc.

We will develop Ansible scripts to simulate the use of a jump box or bastion host. A jump box serves as an intermediary server that can be used to gain access to web servers within a secured network. It provides better security and reduces attack surface.

## Objectives
- Install and configure Ansible client to act as a jump server or bastion host

- Create a simple Ansible playbook to automate servers configuration.

## Prerequisites
- Basic AWS knowledge
- Ansible
- Git

## Step 1 - Install and Configure Ansible on Ec2 instance

1. Update the `Name` tag on your `Jenkins` EC2 instance to `Jenkins-Ansible`. We will use this server to run playbooks.

![](/images/Screenshot%20(665).png)

2. In your GitHub account, create a new repository named `ansible-config-mgt`. This repository will store your Ansible configurations, playbooks, and inventory files.

3. Install Ansible on your Jenkins-Ansible ec2 instance. Update your package index:

```bash
sudo apt update
```

4. Install Ansible:

```bash
sudo apt install ansible
```
![](/images/Screenshot%20(642).png)

5. Verify the installation by checking the Ansible version:

```bash
ansible --version
```
![](/images/Screenshot%20(643).png)

- ## Step 2 Configure jenkins to archive the repository content `ansible-config-mgt`.

- Configure a Webhook in GitHub and set the webhook to trigger `ansible` build.
On `ansible-config-mgt` repository, select Settings > Webhooks > Add webhook
![](/images/Screenshot%20(649).png)

We will configure a jenkins job to trigger from a github webhook set to trigger `ansible-job` build. Create a freestyle job in jenkins and click **Configure** Add the following configurations:

  - Configure Build Triggers: Github hook trigger for GITScm polling
  ![](/images/Screenshot%20(645).png)

  - Configure Post-build actions :Archive the artifact
  ![](/images/Screenshot%20(646).png)

We will test our configuration by making a little change to the Readme file in the `ansible-config-mgt` repository.
  - Ensure the build artifacts are saved in the folder:

```bash
/var/lib/jenkins/jobs/ansible-job/builds/[build_number]/archive/
```
![](/images/Screenshot%20(666).png)

Also, Every time you Stop/Start your Jenkins-Ansible server, you hsve to configure github webhook to a new IP address. In order to avoid it, It only makes sense to allocate an Elastic IP to your Jenkins-Ansible Ec2 instance.
- We will allocate an elastic IP to the `jenkins-ansible` server by following these steps:
  - Navigate to the EC2 Dashboard

  - In the console, find and click on EC2 under the "Services" menu.

  - In the left navigation pane, click on **Elastic IPs**.

  - Click on the **Allocate Elastic IP address button**.
![](/images/Screenshot%20(647).png)

   - Associate the **Elastic IP address to the jenkins-ansible ec2**
![](/images/Screenshot%20(648).png)   

## Step 3 Prepare your development environment using visual Studio Code.
We will use visual studio code to prepare our dev environment.
- Connect your visual studio code to your Github repository.
- Clone the `ansible-config-mgt` repository to the the `jenkins-ansible-server`:

```bash
git clone https://github.com/kenneth-stack/ansible-config-mgt.git
```
![](/images/Screenshot%20(650).png)

## Step 4 Begin Ansible development
- We will create a new branch named `feat/prj-11-ansible-config` that we will use for developing a new feature. From the command line enter:

```sh
git checkout -b feat/prj-11-ansible-config
```
- Checkout the newly created feature branch to your local machine and start building your code and directory structure

```bash
git fetch
git checkout feature/prj-11-ansible-config
```
- Create a directory and name it `playbooks` - it will be used to store all your playbook files.

```bash
mkdir playbooks
```
- Create a directory and name it `inventory` - it will be used to keep your hosts organised

```bash
mkdir inventory
```
- Within the playbooks folder, create your first playbook, and name it common.yml

```bash
touch playbooks/common.yml
```
- Within the inventory folder, create an inventory file (.yml) for each environment (Development, Staging, Testing and Production) dev, staging, uat, and prod respectively.

```bash
touch inventory/dev.yml inventory/staging.yml inventory/uat.yml inventory/prod.yml
```
These inventory files use .ini languages style to configure Ansible hosts.
![](/images/Screenshot%20(653).png)

## Step 5 Set up Ansible inventory

In ansible, we plan to execute linux commands on remote hosts. Ansible will use its default port `22` to SSH into target servers from the `jenkins-ansible-server` host. We will be import our existing key using the `ssh-agent` unto the `jenkins-ansible-server`

To learn how to setup SSH agent and connect VS Code to your Jenkins-Ansible instance, please see this video:

- For Windows users - [ssh-agent on windows](https://www.youtube.com/watch?v=OplGrY74qog)
- For Linux users - [ssh-agent on linux](https://www.youtube.com/watch?v=OplGrY74qog)

Follow the steps:
  - Setup the SSH Agent (Run this on your local machine):

```bash
    # Start the SSH agent
    eval "$(ssh-agent -s)"

    # Verify the agent is running
    echo $SSH_AGENT_PID
    
    # Add your SSH key If you have an existing key (private key)

    ssh-add ~/.ssh/private_key

    # Or optionally generate a new key if needed
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    ssh-add ~/.ssh/id_rsa
    
    # Verify the added keys:
    
    ssh-add -l
```

  - Test the connection by SSH into the `jenkins-ansible-server` with the SSH-agent from your local machine.

```bash
  ssh -A ubuntu@<jenkins-ansible-server-public-ip>
 ```
  The -A flag explicitly enables agent forwarding, allowing the Jenkins-Ansible server to use your local machine's SSH keys when connecting to other servers. 

  - Verify the SSH agent forwarding on the `jenkins-ansible` server

```bash
   # On jenkins-ansible: Verify SSH agent is forwarded
   echo $SSH_AUTH_SOCK    # Should show a socket path
   ssh-add -l            # Should show your key(s)
```
 *Troubleshooting*

  - Ensure your security group settings on the instance allow SSH access on port 22 from your host IP
  - ensure you connect to your instance form your local host and accept the host key fingerprint, first.
  - If you have multiple ssh keys in your system, ensure that the key you intend to use is indeed added.
  - Also set the correct permissions for .pem keys from AWS (`chmod 400 ~/path-to-key`)


  **Connect the key to VS-code**:

  We can optionally connect VS code remotely to the `jenkins-ansible` server if we want to use VS code to edit files on the `jenkins-ansible` server:

  Follow the steps:

  - Install **Remote-SSH** extension in VS code
  - Press F1 or Ctrl+Shift+P and type "Remote-SSH: Connect to Host"
  - Enter: ubuntu@<jenkins-ansible-public-ip> 
  - or (first) create edit the `.ssh/config` file. Add the following content: Afterwards connect to the host via vscode

```bash
    # Replace the placeholders

    Host jenkins-machine-ip
    User ec2-user  # Replace with the appropriate username if different
    ForwardAgent yes
```

**Update `inventory/dev.yml` file**

 Update the `inventory/dev.yml` file with the connection information of the NFS server, the three webservers, the DB server and the load balancer server:

 ```bash
 [nfs]
 <NFS-Server-Private-IP-Address> ansible_ssh_user=ec2_user

 [webservers]
 <webserver1-Private-IP-Address> ansible_ssh_user=ec2_user
 <webserver2-Private-IP-Address> ansible_ssh_user=ec2_user

 [db]
 <Database-Server-Private-IP-Address> ansible_ssh_user=ec2_user

 [lb]
 <loadbalancer-Server-Private-IP-Address> ansible_ssh_user=ubuntu
 ```

 Replace the placeholders as follows:

 ```bash
 [nfs]
172.31.32.173 ansible_ssh_user=ec2-user

[webservers]
172.31.6.35 ansible_ssh_user=ec2-user
172.31.2.9 ansible_ssh_user=ec2-user

[db]
172.31.32.34 ansible_ssh_user=ubuntu

[lb]
172.31.35.177 ansible_ssh_user=ubuntu
 ```
![](/images/Screenshot%20(654).png)

## Step 6 Create a Common Playbook
 We will use the `common.yml` playbook to write configuration for repeatable and re-usable tasks that can be run on multiple machines:

 Edit the `playbook/common.yml` file with the code below:

 ```yaml
 ---
 - name: update web and nfs servers
  hosts: webservers, nfs
  become: yes
  tasks:
    - name: ensure wireshark is the latest version
      yum:
        name: wireshark
        state: latest

- name: update lb and db server
  hosts: lb, db
  become: yes
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is the latest version
      apt:
        name: wireshark
        state: latest
 ```
![](/images/Screenshot%20(655).png)

 *Hint*: Ensure to align the yaml file properly.

The ansible playbook automates tasks on the servers.

The playbook consists of two plays:
Play 1 updates web and NFS which are based on RHEL. It uses elevated privileges (become: yes) and installs/updates Wireshark to the latest version using yum.

Play 2 update LB server and DB. It uses elevated privileges (become: yes) and updates the apt repository cache. It also installs/updates Wireshark to the latest version using apt.

We will later update the `common.yml` to include additional tasks in another file named `common2.yml`

 Include tasks to 
 - Create a directory and a file inside it
 - Change timezone on all the servers
 - Run some shell scripts. Create shell scripts

Create a `common2.yml` file and paste the following content to the `common2.yml` file

**Updated `common2.yml` file**:
```yaml
---
- name: Update and configure servers
  hosts: all  # Target all servers
  become: yes
  vars:
    timezone: 'UTC'  # Change this to your desired timezone
    scripts_dir: '/opt/scripts'  # Directory for shell scripts
    
  tasks:
    # Update package cache based on OS family
    - name: Update apt repo (Ubuntu)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"  # For Ubuntu servers

    - name: Update yum repo (RHEL)
      yum:
        update_cache: yes
      when: ansible_os_family == "RedHat"  # For RHEL servers

    # Install packages based on OS family
    - name: Install packages on Ubuntu
      apt:
        name: wireshark
        state: latest
      when: ansible_os_family == "Debian"

    - name: Install packages on RHEL
      yum:
        name: wireshark
        state: latest
      when: ansible_os_family == "RedHat"

    # Create directory (works on both OS types)
    - name: Create makeshift directory
      file:
        path: ~/makeshift
        state: directory
        mode: '0755'

    - name: Create test.yml file
      copy:
        content: |
          # This is a test file
          key1: value1
          key2: value2
        dest: ~/makeshift/test.yml
        mode: '0644'

    # Set timezone (works on both OS types)
    - name: Set timezone
      timezone:
        name: "{{ timezone }}"

    # Creating and running shell scripts (works on both OS types)
    - name: Create scripts directory
      file:
        path: "{{ scripts_dir }}"
        state: directory
        mode: '0755'

    - name: Create system check script
      copy:
        content: |
          #!/bin/bash
          echo "Running system checks..."
          echo "OS Type: $(cat /etc/os-release | grep PRETTY_NAME)"
          df -h
          free -m
          uptime
        dest: "{{ scripts_dir }}/system_check.sh"
        mode: '0755'

    - name: Create backup script
      copy:
        content: |
          #!/bin/bash
          backup_dir="/backup/$(date +%Y%m%d)"
          mkdir -p $backup_dir
          echo "Starting backup to $backup_dir..."
        dest: "{{ scripts_dir }}/backup.sh"
        mode: '0755'

    - name: Execute system check script
      shell: "{{ scripts_dir }}/system_check.sh"
      register: system_check_output

    - name: Display system check results
      debug:
        var: system_check_output.stdout_lines
```
![](/images/Screenshot%20(661).png)

**Explanation of the script functionality**
This Ansible playbook is designed to update, configure, and perform system checks on all targeted servers, regardless of their operating system (Ubuntu/Debian or RHEL/CentOS). The playbook first updates the package cache and installs/updates Wireshark using either `apt` or `yum`, depending on the server's OS family. It then creates a makeshift directory and test YAML file, sets the system timezone, and creates a scripts directory.

The playbook also creates two shell scripts: `system_check.sh` and `backup.sh`. The `system_check.sh` script runs system checks, including OS type, disk space, memory usage, and uptime, and displays the results. The `backup.sh` script creates a backup directory and starts a backup process, although it currently only includes a placeholder message. The playbook uses conditional statements to apply tasks based on the server's operating system family, ensuring compatibility across different environments.

We will also need to create an variable file named `all.yml` in a folder named `group_vars`

```bash
mkdir group_vars
touch all.yml
```
- Paste the following content to the `all.yml`

```sh
timezone: 'America/New_York'  # Change timezone as needed
scripts_dir: '/opt/scripts'   # Change scripts directory if desired
```
![](/images/Screenshot%20(662).png)

## Step 7 - Update GIT with the latest code.

Collaboration in a DevOps environment involves using Git for version control, where changes are reviewed before merging into the main branch.

- Check the Status of Modified Files:
```bash
git status

```
- Add Files to Staging Area: Specify files to be committed, or use . to stage all changes:
```bash
git add .
```

- Commit the Changes with a Message: Include a clear and descriptive commit message:
```bash
git commit -m "Add playbook with initial configuration tasks"
```

- Push your changes to the feature branch:
```bash
git push origin feature/prj-11-ansible-config-mgt
```

- Create a Pull Request (PR)
* Navigate to Your Repository on GitHub and locate your branch.
* Click on Pull Request to compare changes between the feature branch and main.
* Add a Title and Description for the PR and submit it for review.

![](/images/Screenshot%20(656).png)

![](/images/Screenshot%20(657).png)

- Review and Merge the PR
* Switch roles to review the PR. Check the code for accuracy and adherence to standards. If satisfied:
* Approve the PR and Merge it into the main branch.
* Once merged, delete the feature branch if no longer needed.

![](/images/Screenshot%20(658).png)

- On your local machine, switch back to main, pull the latest changes, and confirm the merge:
   
```bash
git checkout main
git pull origin main
```
![](/images/Screenshot%20(659).png)

### After the changes are merged, Jenkins will automatically trigger a build, archiving the files in the following directory:
```bash
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```
![](/images/Screenshot%20(663).png)

## Step 8 Run Ansible test
We will verify if the playbook works :

Change directory to `ansible-config-mgt`:
```sh
cd ansible-config-mgt
```

Run the playbook command:

```sh
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```
![](/images/Screenshot%20(667).pngx)


This command does the following:
    * Uses `-i inventory/dev.yml` to specify the inventory file for the development environment.
    * Targets the `playbooks/common.yml` playbook.
    * Note: Make sure you’re in the `ansible-config-mgt` directory before running the command.

# Key Learnings

1. Fundamentals of Ansible Configuration Management: Learned how to set up and deploy configurations across multiple servers with Ansible.

2. Inventory Management: Set up an inventory file to manage hosts in different groups (e.g., web, nfs, db servers).

3. Error Troubleshooting: Developed troubleshooting skills for Ansible-related SSH issues and YAML syntax errors

4. Configuring SSH: Configured ssh access and handled key-based authentication, ensuring secure connections to remote servers.

5. Ansible Playbook Structure: Gained a strong understanding of structuring playbooks to automate common tasks, manage dependencies, and execute sequential commands.

## Challenges and Solutions

### Challenge: SSH Authentication Issues
Solution: Verified file paths and permissions for the SSH keys and ensured the IdentityFile path in the Ansible inventory and SSH config was accurate and correctly formatted.

### Challenge: "Permission Denied" errors on remote servers
Solution: Confirmed that the correct user (e.g., ubuntu, ec2-user) was set up on each server. This was resolved by specifying the exact username in the Ansible inventory and ensuring the keys had the correct permissions.

### Challenge: Issues with YAML Formatting in Inventory Files
Solution: Ensured proper YAML syntax, particularly around indentation and structure, which is critical in Ansible files. Simplified formatting and used tools to validate YAML files before deployment.

### Challenge: Host Key Verification Failures
Solution: Used ssh-keygen -R <host> to clear known hosts entries, added host keys permanently to avoid prompting, and used the -i option to specify the key file directly during testing.

# Conclusion

We have successfully set up our infrastructure to automate configuration mangaement tasks with ansible.





