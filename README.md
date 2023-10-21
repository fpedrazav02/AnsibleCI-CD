# AnsibleCI-CD

## Description
You are a DevOps engineer at XYZ Ltd. Your company is working on a Java application and wants to automate WAR file artifact deployment so that they don’t have to perform WAR deployment on Tomcat/Jetty web containers. Automate Ansible integration with Jenkins CI server so that we can run and execute playbooks to deploy custom WAR files to a web container and then perform restart for the web container.

## Steps to Perform:
- Configure Jenkins server as Ansible provisioning machine
- Install Ansible plugins in Jenkins CI server
- Prepare Ansible playbook to run Maven build on Jenkins CI server
- Prepare Ansible playbook to execute deployment steps on the remote web container with restart of the web container post deployment

# Solution

# 1) Check if Ansible is installed

```shell
ansible --version
```
![alt text](.\img/ansversion.png)

We then install it.

```shell
sudo apt install ansible
```
Then we check installation

![alt text](.\img/ansversion2.png)

Then we can modify our `/etc/ansible/hosts` file in order to create a group of worknodes

![alt text](.\img/hosts.png)

After configuring our inventory file we check if it is working and if we have connectivity with our worker nodes.

![alt text](.\img/workerconnectcheck.png)

> Since both failed it means we need to generate a new ssh key and we will use it through a new ansisuser. 

We will do the same three steps on each Node.

### 1) Add ansiuser
We will create ansiuser, the user which ansible will use.
```bash
sudo su - #Use root perms

adduser ansiuser
```
### 2) Modify /etc/ssh/sshd_config
In order to connect without needing password we will need to edit sshd daemon config files as well as the sudoers file. 

```bash
sudo vim /etc/ssh/sshd_config
```
![Alt text](.\img/nopasswd.png)

Then we restart the service
```
sudo service sshd restart
```
### 3) Finally add perms on the sudoers
```bash
sudo vim /etc/sudoers
```
![Alt text](.\img/sudoers.png)

After adding it to the worker nodes we will create and copy a new ssh key from the AMC. 
![Alt text](.\img/sshkeygen.png)

We then copy it to the worker nodes

```bash
ssh-copy-id -i ansiuser@hostip
```
![Alt text](.\img/keyadd.png)

Then we can finally check for the connection on our **worknodes**.
![Alt text](.\img/pongsuccess.png)
> We hard reseted lab. Ips may have changed.

# PlayBook Creation
First we can create a PlayBook to install all needed dependencies and build our **War** file. We will need to install.

- Git
- Maven

We can create a simple playbook that checks for each dependency and it's version. Then, we store the output in a variable and install the dependency if needed.

**Example of checking and installing a dependency:**
```yaml
 - name: Check if Git is installed
      command: git --version
      register: git_check
      ignore_errors: yes
 - name: Install Git
      package:
        name: git
        state: present
      when: git_check.rc != 0
``` 
Addiontonally, we need to add to our PlayBook. 

- Ability to clone the repository
- Build our War file with Maven

Our Java WebApp is part of ***Sonals repo***.
You may find the full Java WebApp [here](https://github.com/Sonal0409/DevOpsCodeDemo).


```yml
---
- name: Install Dependencies
  hosts: worknodes
  become: true
  vars: 
  tasks:
    - name: Update Repository
      command: sudo apt-update

    - name: Check if Git is installed
      command: git --version
      register: git_check
      ignore_errors: yes
    - name: Check if Maven is installed
      command: mvn --version
      register: maven_check
      ignore_errors: yes

    - name: Install Git
      package:
        name: git
        state: present
      when: git_check.rc != 0

    - name: Install Maven
      package:
        name: maven
        state: present
      when: maven_check.rc != 0

    - name: Clone the repository
      git: repo=https://github.com/Sonal0409/DevOpsCodeDemo.git dest=/tmp/code

    - name: Build with Maven
      command: chdir=/tmp/code mvn package
```
Before deploying it via Jenkins. We should try it.

```
ansible-playbook -i /home/aniuser/inventory InstallationPlayBook.yml
```
![Alt text](.\img/pbruntest.png)

We can check as well if the files were created on the node.

```
ansible -i /home/aniuser/inventory worknodes -m command -a "ls -s /tmp/code/target"
```
![Alt text](.\img/ansifiles.png)
---
Since we have our CI part with a PlayBook. We can create another playbook to copy the files needed and building a docker image. After it, it will deploy it.

We will use the same dockerfile as last time.

```dockerfile
FROM tomcat:9
ADD addressbook.war /usr/local/tomcat/webapps
CMD ["catalina.sh","run"]
EXPOSE 8080
```
The PlayBook looks like this.

```yml
---
- name: CI/CD PlayBook
  hosts: worknodes
  vars:
  become: true
  tasks:
    - name: Start Docker Service
      service: name=docker state=started
    - name: Copy War file to dockerfiles dir
      copy: src=/tmp/code/target/addressbook.war dest=/tmp/code remote_src=yes
    - name: Build Docker Image
      command: chdir=/tmp/code docker build -t projectimage .
    - name: Run Docker Image
      command: docker run -d -P projectimage
```
Execution functions perfectly
![Alt text](.\img/dockerplayb.png)

When checking both containers and their runnign status we see they are working.
![Alt text](.\img/runcont.png)

# Jenkins PlayBook Setup

Firstly we check for Java and if it is install
```
java -version
```
Since we do not have it installed we can install it with
```
apt install default-jre
```

> Since we were having troubles installing jenkins. We decided to install another version on a virtual machine.

![Alt text](.\img/jenkinsfail.png)

Without a handshake verification is not posible to install jenkins. 

We can then continue with the virtual machine one. We will need to install the needed plugins.

![Alt text](.\img/p1.png)
![Alt text](.\img/p2.png)

Once installed we configure the tool.
![Alt text](.\img/p3.png)

## Create Ansible Job
### Create a new Job
We create a pipeline job with then next syntax:

```
pipeline{

    agent any

    stages{

    stage('Clone the playbook repo')
    {
    steps{
        git branch: 'main', url: 'https://github.com/fpedrazav02/AnsibleCI-CD.git'
    }
    }
    stage('Playbook to Build code')
    {

    steps{
        ansiblePlaybook credentialsId: 'ansiblecredentials', disableHostKeyChecking: true, installation: 'myansible', inventory: 'dev.inv', playbook: 'InstallationPlayBook.yml'

    }

    }

    stage('Playbook to deploy code')
    {

    steps{
        ansiblePlaybook credentialsId: 'ansiblecredentials', disableHostKeyChecking: true, installation: 'myansible', inventory: 'dev.inv', playbook: 'dockerCD.yml'
    }
        }
    }
}

```

Finally, we can run the Job.
![Alt text](.\img/run.jpg)
