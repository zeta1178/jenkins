# Jenkins Server Project

### Step by Step Instructions
1) create EC2 t2.medium off ubuntu 
    ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20230516
    ami-053b0d53c279acc90
2) next run steps: https://docs.docker.com/engine/install/ubuntu/
```
#
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
#
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
#
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
#
sudo apt-get update
#
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
#
sudo docker run hello-world
#
sudo systemctl start docker
#
sudo systemctl enable docker
#
sudo usermod -aG docker ubuntu
# 
logout
#
docker ps
# No records listed
# install docker compose plugin
sudo apt-get update
sudo apt-get install docker-compose
docker compose version
#
docker-compose
#
docker pull jenkins/jenkins
# path to docker
docker info | grep -i root
# how much space is docker taking
sudo du -sh /var/lib/docker
#
mkdir jenkins-data
cd jenkins-data
mkdir jenkins_home
#
```
3) Edit the docker-compose.yml
```
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
networks:
  net:  
```
4) execute
```
#
sudo chown 1000:1000 jenkins_home -R
#
docker-compose up -d
#
docker logs -f jenkins
#
```
5) Grab the passwrd on screen
```
# Example password
3510ddda1b2f44ba81ed37b3775fd572
```
6) grab the Public IPv4 DNS , and append ':8080' to the end
```
# Example
http://ec2-100-26-1-175.compute-1.amazonaws.com:8080/
# Select install suggested plugins
# Wait for plugins to install
# Add admin user details
# Logout then backin as Admin
#
```
7) execute
```
# stops the services 
docker-compose stop 
# starts the service
docker-compose start
# restarts service
docker-compose restart jenkins
#
```
8) execute
```
#
mkdir centos7
cd centos7
touch Dockerfile
#
```
9) edit Dockerfile in the centos7 folder
```
FROM centos:7

RUN yum -y install openssh-server

RUN useradd remote_user && \
        echo "1234" | passwd remote_user --stdin && \
        mkdir /home/remote_user/.ssh && \
        chmod 700 /home/remote_user/.ssh

COPY remote-key.pub /home/remote_user/.ssh/authorized_keys

RUN chown remote_user:remote_user -R  /home/remote_user/.ssh/ && \
        chmod 600 /home/remote_user/.ssh/authorized_keys

RUN /usr/sbin/sshd-keygen

CMD /usr/sbin/sshd -D
```
10) execute
```
# create a ssh key pair
ssh-key-gen -f remote-key
#
```
11) modify docker-compose.yml 
```
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
  remote_host:
    container_name: remote-host
    image: remote-host
    build:
      context: centos7
    networks:
      - net
networks:
  net: 
```
12) execute
```
#
docker-compose build
#
docker-compose up -d
#
```
13) ssh from jenkins container to remote-host
```
#
docker exec -ti jenkins bash
#
ssh remote_user@remote_host
# say yes to add fingerprint
```
14) execute
```
# copy ssh key to the jenkins container
docker cp remote-key jenkins:/tmp/remote-key
#
docker exec -ti jenkins bash
#
cd tmp
#
ssh -i remote-key remote_user@remote_host
```
15) in the web app, go to Manage Jenkins, Plugins, find SSH, install, restart
16) execute
```
# restart the service
docker-compose start
```
17) in the web app, go to Manage Jenkins, Credentials, Global, Add Credentials, pick SSH Username and private key
18) Username=remote_user , select Private Key
19) in EC2, cd jenkins-data, cd centos7
20) execute
```
#
cat < remote-key
# copy contents to clipboard
# paste in the web app
```
21) in the web app, go to Manage Jenkins, System, scroll down to SSH, select credential created in above steps, use Port 22, use Hostname= remote_host, check connection, and Save
22) Create item
```
# create new item, remote-task, build step, execute shell on remote host
#
NAME=mike
echo "Hello, $NAME" > /tmp/remote-file
# save , build, 
```
23) execute
```
#
docker exec -ti remote-host bash
#
cd tmp
#
ls
vi remote-file
#
```
24) in the web app, go to Manage Jenkins, Credentials, Global, new Credentials, Secret Text, MYSQL_PASSWORD = 1234
25) in the web app, go to Manage Jenkins, Credentials, Global, new Credentials, Secret Text, AWS_SECRET_KEY
26) modify docker-compose.yml
```
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
  remote_host:
    container_name: remote-host
    image: remote-host
    build:
      context: centos7
    networks:
      - net
  db_host:
    container_name: db
    image: mysql:8.0
    environment:
      - "MYSQL_ROOT_PASSWORD=1234"
    volumes:
      - $PWD/db_data:/var/lib/mysql
    networks:
      - net
networks:
  net:  
```
27) execute
```
#
docker-compose up -d
# check status of MYSQL 
docker logs -f db
#
docker exec -ti db bash
mysql -u root -p
# Enter 1234
# quit, exit
```
28) execute
```
# 
cd centos7
vi Dockerfile
#
FROM centos:7

RUN yum -y install openssh-server

RUN useradd remote_user && \
        echo "1234" | passwd remote_user --stdin && \
        mkdir /home/remote_user/.ssh && \
        chmod 700 /home/remote_user/.ssh
COPY remote-key.pub /home/remote_user/.ssh/authorized_keys

RUN chown remote_user:remote_user -R  /home/remote_user/.ssh/ && \
        chmod 600 /home/remote_user/.ssh/authorized_keys

RUN /usr/sbin/sshd-keygen

RUN yum -y install mysql 

RUN yum -y install epel-release && \
    yum -y install python3-pip && \
    pip3 install --upgrade pip && \
    pip3 install awscli


CMD /usr/sbin/sshd -D
```
29) execute
``` 
cd ..
docker-compose build
```
30) execute 
```
docker-compose up -d
docker exec -ti remote-host bash
aws
mysql (sockets error, good)
```
31) execute
```
# login to remote host
docker exec -ti db bash
mysql -u root -p
# Add edit, once logged into mysql container as root
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '1234';
# quit, exit
docker exec -ti remote-host bash
mysql -u root -h db_host -p
show databases;
create database testdb;
use testdb;
create table info (name varchar(20), lastname varchar(20), age int(2));
show tables;
desc info;
insert into info values ('mike', 'cruz', 44);
select * from info;
```
32) execute
```
# locate the jenkins-data folder 
touch aws-s3.sh
# add the following
#/bin/bash

DATE=$(date +%H-%M-%S)
BACKUP=db-$DATE.sql

DB_HOST=$1
DB_PASSWORD=$2
DB_NAME=$3
AWS_SECRET=$4
BUCKET_NAME=$5

mysqldump -u root -h $DB_HOST -p$DB_PASSWORD $DB_NAME > /tmp/$BACKUP && \
export AWS_ACCESS_KEY_ID=AKIAJ6DYDN73DAFICOGQ && \
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET && \
echo "Uploading your $BACKUP backup" && \
aws s3 cp /tmp/$BACKUP s3://$BUCKET_NAME/$BACKUP
rm -f /tmp/$BACKUP
#
chmod +x aws-s3.sh
docker cp aws-s3.sh remote-host:/tmp/aws-s3.sh
#
```
33) in the web app, go to new item, label "backup-to-aws", freestyle, save
34) check this is parameterized, follow screen, build environment use secret text, follow screen
35) Configure jenkins project
![Parameters](1a.jpg)
36) Environment Variables
![Environment Variables](1.jpg)
37) create build step, execute shell on remote host 
```
/tmp/aws-s3.sh $MYSQL_HOST $MYSQL_PASSWORD $DATABASE_NAME $AWS_SECRET_KEY $AWS_BUCKET_NAME
```
38) modify docker-compose.xml, to persist script for aws
```
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
  remote_host:
    container_name: remote-host
    image: remote-host
    build:
      context: centos7
    volumes:
      - $PWD/aws-s3.sh:/tmp/aws-s3.sh
    networks:
      - net
  db_host:
    container_name: db
    image: mysql:8.0
    environment:
      - "MYSQL_ROOT_PASSWORD=1234"
    volumes:
      - $PWD/db_data:/var/lib/mysql
    networks:
      - net
networks:
  net: 
```
39) execute
```
docker rm -fv remote-host
docker-compose up -d
```
40) execute
```
# set folder jenkins-data
mkdir jenkins-ansible
cd jenkins-ansible
touch Dockerfile
```
41) edit Dockerfile inside jenkins-ansible
```
FROM jenkins/jenkins

USER root

RUN apt-get update && apt-get install python3-pip -y && \
    pip3 install ansible --upgrade

USER jenkins
```
42) return to jenkins-data folder, edit docker-compose.yml
```
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins-ansible
    build:
      context: jenkins-ansible     
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
  remote_host:
    container_name: remote-host
    image: remote-host
    build:
      context: centos7
    volumes:
      - $PWD/aws-s3.sh:/tmp/aws-s3.sh
    networks:
      - net
  db_host:
    container_name: db
    image: mysql:8.0
    environment:
      - "MYSQL_ROOT_PASSWORD=1234"
    volumes:
      - $PWD/db_data:/var/lib/mysql
    networks:
      - net
networks:
  net:
```
43) execute
```
#rebuild
docker-compose build
docker-compose up -d
docker exec -ti jenkins bash
```
44) execute
```
#return to jenkins-data folder
mkdir jenkins_home/ansible
#return to jenkins-data folder
cp centos7/remote-key jenkins_home/ansible/
#return to jenkins-data folder
cd jenkins-ansible
cp ../centos7/remote-key .
vi hosts
#
[all:vars]

ansible_connection = ssh

[test]

test1 ansible_host=remote_host ansible_user=remote_user ansible_private_key_file=/var/jenkins_home/ansible/remote-key
#copy hosts to jenkins
cp hosts ../jenkins_home/ansible/
#
docker exec -ti jenkins bash
cd $HOME
cd ansible
ansible -i hosts -m ping test1
# No errors
```
45) execute
```
#return to jenkins-data folder
cd jenkins-ansible
touch play.yml
#
- hosts: test1
  tasks:

    - shell: echo Hello World from Ansible > /tmp/ansible-file
#from jenkins ansible run the follow or create jenkins pipeline
ansible-playbook -i hosts play.yml
#
```
46) Ansible Pipeline
![Ansible Pipeline](3.jpg)
47) execute
```
#Edit play.yml
- hosts: test1
  tasks:
   - debug:
       msg: "{{ MSG }}"
```
48) Ansible Pipeline
![Ansible Pipeline](4.jpg)
49) Ansible Pipeline 2
![Ansible Pipeline](5.jpg)
50) execute
```
#create web  foler and sub folders
#edit dcoker-compose.yml
#
version: '3'
services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
    networks:
      - net
  remote_host:
    container_name: remote-host
    image: remote-host
    build:
      context: centos7
    volumes:
      - $PWD/aws-s3.sh:/tmp/aws-s3.sh
    networks:
      - net
  db_host:
    container_name: db
    image: mysql:8.0
    environment:
      - "MYSQL_ROOT_PASSWORD=1234"
    volumes:
      - $PWD/db_data:/var/lib/mysql
    networks:
      - net
  web:
    container_name: web
    image: ansible-web
    build:
      context: jenkins-ansible/web
    ports:
      - "80:80"
    networks:
      - net
networks:
  net: 
#
```
51) execute
```
docker-compose build
docker-compose up -d
docker exec -ti web bash
cd /var/www/html
touch index.php
vi index.php
#
<?php

// Show all information, defaults to INFO_ALL
phpinfo();

// Show just the module information.
// phpinfo(8) yields identical results.
phpinfo(INFO_MODULES);

?>
#check URL of jenkins server  with port 80
#add file table.j2
docker cp table.j2 web:/var/www/html/index.php
#
```