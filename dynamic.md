# Ansible Dynamic Assignments- Community Roles 

While static assignments use `import` module as seen in the [previous project](https://github.com/kenneth-stack/ansible-config-mgt/blob/main/refactoring.md), the `include` module is used for dynamic assignments. It is recommended to use static assignments for playbooks because it is more reliable. 

In this long task 😊, we will introduce dynamic assignment into our structure. We also make use of ansible community roles to make our configuration modular and reusable. I include the steps I took to troubleshoot errors and get it working. When using ansible, like several other devOps tools, you may need to iterate several times to get your app working. Let's begin...

## Set-up Infrastructure

- Create a new branch in the `ansible-config-mgt` repo named `dynamic-assignments`

```sh
# Create branch locally in the jenkins-ansible server
git checkout -b feat/dynamic-assignments

# push the branch to the remote repo
git push -u origin feat/dynamic-assignments
```
![](/images3/Screenshot%20(696).png)

Create a new folder, name it `dynamic-assignments`.
Then inside this folder, create a new file and name it `env-vars.yml`. We will instruct `site.yml` to `include` this playbook later. For now, let us keep building up the structure.

```bash
mkdir dynamic-assignments
touch dynamic-assignments/env-vars.yml
```
![](/images3/Screenshot%20(697).png)

Your GitHub shall have following structure by now.

__Note__: Depending on what method you used in the previous project you may have or not have `roles` folder in your GitHub repository - if you used `ansible-galaxy`, then `roles` directory was only created on your `Jenkins-Ansible` server locally. It is recommended to have all the codes managed and tracked in GitHub, so you might want to recreate this structure manually in this case - it is up to you.

```css
├── dynamic-assignments
│   └── env-vars.yml
├── inventory
│   └── dev
    └── stage
    └── uat
    └── prod
└── playbooks
    └── site.yml
└── roles (optional folder)
    └──...(optional subfolders & files)
└── static-assignments
    └── common.yml
```

- Create a folder of environment variables named `env-vars` in the root directory. Each environment will have separate variables hence, create yaml files within the folder to store environment-specific variables named`dev.yml`, `stage.yml`, `uat.yml` and `prod.yml` respectively.

![](/images3/Screenshot%20(698).png)

The following yaml configuration consists of
- `with_first_found`: This is a lookup plugin that searches for the first file it finds from the list. It will:
  - Look through the files in order: dev.yml → stage.yml → prod.yml → uat.yml
  - Use the first file it finds in the `env-vars`  directory
  - Load those variables into the playbook


- {{ playbook_dir }}/../env-vars: This path tells Ansible to look in the env-vars directory that's one level up from the playbook location. 

Notes on ansible special variables can be found in the [documentation](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html). Also , details about looping over lists of items is [here](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html#standard-loops)

```yml
## Correct syntax. Check explanations in the troubleshooting section
---
- name: Collate variables from environment-specific file
  include_vars:
    file: "{{ item }}"
  with_first_found:
    - files:
        - "{{ inventory_file | basename | splitext | first }}.yml"
        - dev.yml
        - staging.yml
        - prod.yml
        - uat.yml
      paths:
        - "{{ playbook_dir }}/../env-vars"
  tags:
    - always
```
Paste the above instruction into the `env-vars.yml` file
![](/images3/Screenshot%20(699).png)


## Step 1 Update `site.yml` with dynamic assignments
Next, we will update the `site.yml` with dynamic assignments.

```
include: ../dynamic-assignments/env-vars.yml
```

```sh
---
# Play 1: Loads dynamic variables for all hosts
- name: Include dynamic variables
  hosts: all
  tasks:
    - name: Load dynamic variables
      include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

# Play 2: Imports common configuration for all hosts
- name: Import common playbook
  import_playbook: ../static-assignments/common.yml

# Play 3: Specific configuration for UAT webservers only
- name: Configure UAT webservers
  hosts: uat-webservers
  tasks:
    - name: Import UAT specific tasks
      import_tasks: ../static-assignments/uat-webservers.yml

```
Here is what the `site.yml` looks like now:

![update site.yml](/images3/Screenshot%20(700).png)

To test the configuration, you may need to start the remaining webservers, now, if they are not already started.

Change directory to the `ansible-config-mgt` root directory and run:

```sh
ansible-playbook -i inventory/dev.yml playbooks/site.yml
```

## Step 2 Creating community role for the MySQL db

We will create a role for the MySQL db that installs the MySQL package, creates a database and configure users. Rather than, starting from scratch we will use a pre-existing production ready role from the ansible galaxy.

First we will download the Mysql ansible role. Available community roles can be found in the [ansible galaxy website](https://galaxy.ansible.com/ui/)

For the MySQL role we use a popular one by [geerlingguy](https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/mysql/)

We already have git installed in our machine as well as the git initialised `ansible-config-mgt` directory. We will create a new branch named `roles-features` and switch to it.

Navigate to the roles directory and install the mySQL role with the following command:

```sh
ansible-galaxy install geerlingguy.mysql
```

**Community role installed successfully**

Rename the created folder to mysql:

```sh
mv geerlingguy.mysql/ mysql
```

Edit the roles configuration to use the correct credentials for MySQL required for the `tooling` website.

We can use the community role to:
- Manage our existing database
- Create databases for different environments.

To apply the community role to our use case, managing our existing database, first, we will define environment variables.

    - Define environment-specific MySQL database credentials in each of your `env-vars` files (like `dev.yml`, `prod.yml`, etc.)
  For instance, the `dev.yml` example:

  ```yaml
  # MySQL configuration for the development environment
mysql_root_username: "root"
mysql_root_password: "Passw0rd123#"

# Define databases and users to be created for the dev environment
mysql_databases:
  - name: "tooling"
    encoding: "utf8"
    collation: "utf8_general_ci"

mysql_users:
  - name: "webaccess"
    host: "%"
    password: "Passw0rd321#"
    priv: "my_dev_database.*:ALL"

  ```
  ![](/images3/Screenshot%20(701).png)
  Replace the database name, mysql_users name and passwords as appropriate.

Update similarly for `prod.yml`, `staging.yml`, and `uat.yml` with specific database names and credentials for each environment.

Next, we update the inventory files with environment-specific variables. Each inventory file (like `inventory/dev.yml`, `inventory/prod.yml`), include the path to the relevant environment variables file.

**Example (`inventory/dev.yml)**

```yaml
[nfs]
172.31.32.173 ansible_ssh_user=ec2-user

[webservers]
172.31.6.35 ansible_ssh_user=ec2-user
172.31.2.9 ansible_ssh_user=ec2-user

[db]
172.31.32.34 ansible_ssh_user=ubuntu

[lb]
172.31.35.177 ansible_ssh_user=ubuntu

[db:vars]
env_vars_file=../env-vars/dev.yml
```
![](/images3/Screenshot%20(702).png)

Next, update playbook configuration in `playbooks/site.yml` including the variable file and specifying the renamed role `mysql` 
```bash
---
# Play 1: Loads dynamic variables for all hosts
- name: Include dynamic variables
  hosts: all
  tasks:
    - name: Load dynamic variables
      include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

# Play 2: Imports common configuration for all hosts
- name: Import common playbook
  import_playbook: ../static-assignments/common.yml

# Play 3: Set up MySQL on database servers
- name: Set up MySQL
  hosts: db
  become: yes
  vars_files:
    - ../env-vars/dev.yml  # Change this to ../env-vars/prod.yml for production, etc.
  roles:
    - mysql  # Ensure this role is installed and named correctly


# Play 4: Specific configuration for UAT webservers only
- name: Configure UAT webservers
  hosts: uat-webservers
  tasks:
    - name: Import UAT specific tasks
      import_tasks: ../static-assignments/uat-webservers.yml

```
![](/images3/Screenshot%20(703).png)

For the `uat-webservers.yml`, we will define the following content. This means the tasks configured in the role will be executed when ansible-playbook is run. 

```sh
- name: Configure uat-webserver
  include_role:
    name: webserver

```
Run the ansible-playbook command from the root directory.
```sh
ansible-playbook -i inventory/dev.yml playbooks/site.yml
```

```sh
---
- name: Collate variables from environment-specific file
  include_vars:
    file: "{{ item }}"
  with_first_found:
    - files:
        - "{{ inventory_file | basename | splitext | first }}.yml"
        - dev.yml
        - staging.yml
        - prod.yml
        - uat.yml
      paths:
        - "{{ playbook_dir }}/../env-vars"
  tags:
    - always
```

This configuration uses file: "{{ item }}" to specify which file to load. It properly structures the file search under `with_first_found`
It adds automatic environment detection using {{ inventory_file | basename | splitext | first }}. It maintains proper YAML hierarchy for the files and paths lists.



Stage and commit the changes to git.
Create a pull request and merge to the main branch

## Step 3 Creating roles for the the load balancers

We want the flexibility to be able to choose between different load balancers: `nginx` or `apache` (remember we previously created a virtual machine for the apache load balancer, in a past project task).
 
We will create different roles for each usecase.

We can choose to develop our own roles or find available ones from the community.

We cannot use both load balancers at the same time so we will include a condition to enable either one applying our variables

**Manual setup of role for loadbalancers** 

Ansible documentation on roles can be found [here](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)

- Create a new branch to work on the manual load balancer setup:

```sh
git checkout -b feat/load-balancer-roles
```

- Navigate to the `roles/` directory and create a `nginx_lb` role for Nginx using `ansible-galaxy` command. I have existing roles called `webserver` (which was created manually) and `mysql` (which was created using community roles) from my previous tasks.

```sh
# list directory content
ls
# create nginx lb directory using ansible-galaxy
ansible-galaxy init nginx_lb

# remove irrelevant directory files: files, tests, vars
rm -rf files tests vars

## Note that this directory manually can be created by running:

# mkdir -p roles/nginx_lb/{tasks,defaults,handlers,templates}
```

- Also create a role for the apache load balancer named `apache_lb`.

```sh
# list directory content
ls
# create nginx lb directory using ansible-galaxy
ansible-galaxy init apache_lb

# remove irrelevant directory files: files, tests, vars
rm -rf files tests vars

## Note that this directory manually can be created by running:

# mkdir -p roles/apache_lb/{tasks,defaults,handlers,templates}
```
![](/images3/Screenshot%20(704).png)

- In the `defaults/main.yml` of both load balancer roles, create default variables to control whether each load balancer is enabled:


**Navigate to `roles/nginx_lb/defaults/main.yml`**

```yaml
# Enable Nginx load balancer
enable_nginx_lb: false
load_balancer_is required: false
```
![](/images3/Screenshot%20(705).png)

**Navigate to `roles/apache_lb/defaults/main.yml`**

```yaml
# Enable Apache load balancer
enable_apache_lb: false
load_balancer_is required: false
```
![](/images3/Screenshot%20(706).png)

Now we already have  nginx load balancer server installed and configured for load balancing in a previous tasks. (check image for the nginx lb running)

To use your this conditional logic in our setup, where the load balancers are already installed in the virtual machines respectively, In `env-vars/[environment.yml]`, In each load balancer role, create tasks to start or stop the respective services based on these variables we have set.

In **`roles/nginx_lb/tasks/main.yml`**

```yaml
---
- name: Ensure Nginx is started for UAT if enabled
  service:
    name: nginx
    state: started
    enabled: true
  when:
    - enable_nginx_lb
    - load_balancer_is_required

- name: Stop Nginx if not required
  service:
    name: nginx
    state: stopped
  when:
    - not enable_nginx_lb
    - load_balancer_is_required

```

In **`roles/apache_lb/tasks/main.yml`**

```yaml
---
- name: Ensure Apache is started for UAT if enabled
  service:
    name: apache2
    state: started
    enabled: true
  when:
    - enable_apache_lb
    - load_balancer_is_required

- name: Stop Apache if not required
  service:
    name: apache2
    state: stopped
  when:
    - not enable_apache_lb
    - load_balancer_is_required
```
![](/images3/Screenshot%20(708).png)
![](/images3/Screenshot%20(707).png)

You will also specify which load balancer to enable in specific environments. For instance,in the UAT environment for the UAT servers, if we want to enable the nginx load balancer, we will set:

**`env-vars/uat.yml`**
```yaml
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true
```
![](/images3/Screenshot%20(709).png)

Update the `playbooks/site.yml` as follows:

```yaml
---

- name: Configure load balancer
  hosts: uat-webservers
  roles:
    - { role: nginx_lb, when: enable_nginx_lb }
    - { role: apache_lb, when: enable_apache_lb }

```
![](/images3/Screenshot%20(710).png)

The playbook snippet:

- configures a load balancer on hosts grouped under `uat-webservers`.
It includes two roles:
  - `nginx_lb`: This role will be executed only if the variable `enable_nginx_lb` is true.
  - `apache_lb`: This role will be executed only if the variable `enable_apache_lb` is true.

- Next, we will test the set up by running:

```
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```
The setup should allow you to toggle between Nginx and Apache load balancers simply by setting the appropriate variables in the environment-specific files. Initially though, I didnt get the expected result. After a couple of iterations and corrections, the errors were resolved.

*Troubleshooting*

1. Verify that the `env-vars/uat.yml` defines the variables correctly
   
  
2. Confirm your `inventory/uat.yml`

`ansible_host` should be correctly set for the webservers and load 
balancers.

3. Review Role Task for Nginx Load Balance in the `roles/nginx_lb/tasks/main.yml`

I modified tasks/main.yml to correct errors, the configuration to include a handler to reload nginx when changes are made. I also create a template file `nginx-lb.conf.j2`

The template file dynamically loads IP addresses from each UAT webserver in the `inventory/uat.yml`

4. Update site.yml

Run playbook:
```sh
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```

We are able to access the uat webservers through the nginx load balancer.

We will also test further by enabling apache while disabling nginx.

Also to do make the site secure using ansible


## Testing apache loadbalancer IP

- Set the `env-vars/uat.yml`
```yml
enable_nginx_lb: false
enable_apache_lb: true
load_balancer_is_required: true

```

- Set the `default/main.yaml`

```sh
# defaults file for apache_lb

enable_apache_lb: false
load_balancer_is_required: false

# Variable for server name and port
apache_server_name: 54.226.233.195  # Server name to use in nginx config
apache_listen_port: 80  # Port to listen on


```

- Configure the task. I modified my yaml file to handle the existing configuration file `webserver-lb.conf` I previously created on the apache_lb server.

```yaml
# tasks file for apache_lb
# tasks file for apache_lb
- name: Check if webserver-lb.conf exists
  stat:
    path: "/etc/apache2/sites-available/webserver-lb.conf"
  register: existing_conf
  when: lb_type == "apache"

- name: Backup existing webserver-lb.conf if it exists
  copy:
    src: "/etc/apache2/sites-available/webserver-lb.conf"
    dest: "/etc/apache2/sites-available/webserver-lb.conf.backup"
    remote_src: yes
  become: true
  when: 
    - lb_type == "apache"
    - existing_conf.stat.exists

- name: Disable existing webserver-lb.conf if it exists
  command: a2dissite webserver-lb.conf
  when: 
    - lb_type == "apache"
    - existing_conf.stat.exists
  notify: restart apache

- name: Ensure Apache is started for UAT if enabled
  service:
    name: apache2
    state: started
    enabled: true
  when:
    - enable_apache_lb
    - load_balancer_is_required
    - lb_type == "apache"

- name: Stop Apache if not required
  service:
    name: apache2
    state: stopped
  when:
    - not enable_apache_lb
    - load_balancer_is_required
    - lb_type == "apache"

- name: Configure Apache Load Balancer if enabled
  template:
    src: "apache-lb.conf.j2"
    dest: "/etc/apache2/sites-available/load-balancer.conf"
    mode: '0644'
  become: true
  when:
    - enable_apache_lb
    - load_balancer_is_required
    - lb_type == "apache"
  notify:
    - restart apache

# Enable the required modules for load balancing
- name: Enable required Apache modules
  command: "a2enmod {{ item }}"
  with_items:
    - proxy
    - proxy_http
    - proxy_balancer
    - lbmethod_byrequests
  become: true
  when: lb_type == "apache"
  notify: restart apache

# enable the load-balancer config
- name: Enable apache load balancer configuration
  command: a2ensite load-balancer.conf
  become: true
  when: lb_type == "apache"
  notify: restart apache
  tags:
    - apache

# disable the default config
- name: Disable default Apache site configuration
  command: a2dissite 000-default.conf
  become: true
  when: lb_type == "apache"
  notify: restart apache
  tags:
    - apache

# Verify configuration
- name: Verify Apache configuration
  command: apache2ctl configtest
  register: apache_config_test
  when: lb_type == "apache"

- name: Display Apache configuration test results
  debug:
    var: apache_config_test.stdout_lines
  when: lb_type == "apache"
```

- Set the `handlers/main.yaml`

```yaml
# handlers file for apache_lb
- name: restart apache
  become: true
  service:
    name: apache2
    state: restarted
  when: lb_type == "apache"

```


- Configure the template named `apache-lb.conf.j2`

```bash
<Proxy "balancer://uat_backend">
    {% for server in groups['uat-webservers'] %}
    BalancerMember "http://{{ hostvars[server].ansible_host }}"
    {% endfor %}
    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName {{ apache_server_name | default('_') }}

    ProxyPreserveHost On
    ProxyPass / balancer://uat_backend/
    ProxyPassReverse / balancer://uat_backend/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

- Check that the `inventory/uat.yml` is well configured with the appropriate variables

```ini
[uat-webservers]

172.31.90.131 ansible_ssh_user='ec2-user' ansible_host='172.31.90.131'
172.31.85.227 ansible_ssh_user='ec2-user' ansible_host='172.31.85.227'


[lb]
172.31.46.249 ansible_ssh_user=ubuntu lb_type=nginx ansible_host='172.31.46.249' # Nginx
172.31.16.59 ansible_ssh_user=ubuntu lb_type=apache ansible_host='172.31.16.59' # Apache
          
```

Replace IP as appropriate for your use case

- Ensure that your `playbooks/site.yml` has the right play for configuring the loadbalancers

```yml
---
# Play 5: Configure load balancers for UAT
- name: Configure load balancer for UAT
  hosts: lb # load balancer defined in inventory/uat.yml
  vars_files:
    - "env-vars/uat.yml"
  roles:
    - { role: nginx_lb, when: load_balancer_is_required and enable_nginx_lb }
    - { role: apache_lb, when: load_balancer_is_required and enable_apache_lb }


```

- Now run your ansible-playbook command (Be ready to correct errors and iterate. 😂)

```sh
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```

Here is the final project structure:


```sh
.
├── README.md
├── ansible.cfg
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
│   ├── dev.yml
│   ├── prod.yml
│   ├── staging.yml
│   └── uat.yml
├── group_vars
│   ├── all.yml
│   └── lb.yml
├── inventory
│   ├── dev.yml
│   ├── prod.yml
│   ├── staging.yml
│   └── uat.yml
├── playbooks
│   └── site.yml
├── roles
│   ├── apache_lb
│   │   ├── README.md
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── apache-lb.conf.j2
│   ├── install_apache_lb
│   │   ├── README.md
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── webserver-lb.conf.j2
│   ├── mysql
│   │   ├── LICENSE
│   │   ├── README.md
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── molecule
│   │   │   └── default
│   │   │       ├── converge.yml
│   │   │       └── molecule.yml
│   │   ├── tasks
│   │   │   ├── configure.yml
│   │   │   ├── databases.yml
│   │   │   ├── main.yml
│   │   │   ├── replication.yml
│   │   │   ├── secure-installation.yml
│   │   │   ├── setup-Archlinux.yml
│   │   │   ├── setup-Debian.yml
│   │   │   ├── setup-RedHat.yml
│   │   │   ├── users.yml
│   │   │   └── variables.yml
│   │   ├── templates
│   │   │   ├── my.cnf.j2
│   │   │   ├── root-my.cnf.j2
│   │   │   └── user-my.cnf.j2
│   │   └── vars
│   │       ├── Archlinux.yml
│   │       ├── Debian-10.yml
│   │       ├── Debian-11.yml
│   │       ├── Debian-12.yml
│   │       ├── Debian.yml
│   │       ├── RedHat-7.yml
│   │       ├── RedHat-8.yml
│   │       └── RedHat-9.yml
│   ├── nginx_lb
│   │   ├── :wq
│   │   ├── README.md
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── nginx-lb.conf.j2
│   └── webserver
│       ├── README.md
│       ├── defaults
│       │   └── main.yml
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
└── static-assignments
    ├── common.yml
    ├── common2.yml
    ├── commondel.yml
    ├── deploy-apache-lb.yml
    └── uat-webservers.yml

```