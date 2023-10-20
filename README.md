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



On first thought we first need to create a playbook which installs all needed dependencies. 

We will do this as a YML.
