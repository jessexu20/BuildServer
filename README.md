# BuildServer Using  Jenkins
##Build up Vagrant Machine
Create a VM to host your build server.  Note, it must be a 64 bit image, otherwise docker will not be supported!

    vagrant init hashicorp/precise64
##Configure VagrantFile to enable public network and forwarded port
	vim Vagrantfile
	config.vm.network "forwarded_port", guest: 6060, host: 6767
	config.vm.network "forwarded_port", guest: 8080, host: 8181

	# Create a private network, which allows host-only access to the machine
	# using a specific IP.
	config.vm.network "private_network", ip: "192.168.33.11"

	# Create a public network, which generally matched to bridged network.
	# Bridged networks make the machine appear as another physical device on
	# your network.
	config.vm.network "public_network"

## Preparing machine for docker

install the backported kernel

    sudo apt-get update
    sudo apt-get -y install linux-image-generic-lts-raring linux-headers-generic-lts-raring

reboot

    sudo reboot

Add the repository to your APT sources

    sudo sh -c "echo deb https://get.docker.com/ubuntu docker main > /etc/apt/sources.list.d/docker.list"

Then import the repository key

    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9

Install docker

    sudo apt-get update
    sudo apt-get install -y lxc-docker

Install some basics    
   
    sudo apt-get -y install git vim


### Creating docker image

Build a docker environment for running build.  Create a "Dockerfile" and place this content inside:

    FROM ubuntu:13.10
    MAINTAINER Chris Parnin, chris.parnin@ncsu.edu
    
    RUN echo "deb http://archive.ubuntu.com/ubuntu saucy main universe" > /etc/apt/sources.list
    RUN apt-get -y update && apt-get -y upgrade
    RUN apt-get install -y wget openjdk-7-jdk curl unzip

    RUN apt-get -y install git
    RUN apt-get -y install maven
    RUN apt-get -y install libblas*
    RUN ldconfig /usr/local/cuda/lib64
    
    ENV JAVA_HOME /usr/lib/jvm/java-7-openjdk-amd64

Build the docker image

    sudo docker build -t ncsu/buildserver .
    
Test the image

    sudo docker images
    sudo docker run -it ncsu/buildserver mvn --version		
### Building

In the host VM, create 'build.sh' and place the following inside: 

    git clone https://github.com/jessexu20/MavenExample
    cd MavenExample/my-app
    mvn clean 
	mvn package

####Execute script

    chmod +x build.sh
    sudo docker run -v /home/vagrant/:/vol ncsu/buildserver sh -c /vol/build.sh
## Install  Jenkins
	wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
	sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
	sudo apt-get update
	sudo apt-get install jenkins
Jenkins can be visited at localhost:8181. In order to do the github hook you have to install the github plugins. 
	Manage Jenkins ->Manage Plugins -> Choose the github plugins
## Configure in Github Repo
In order to enable jenkins features, you have to go to the repo settings.

	Settings->Webhooks & Services -> Add Service -> Select Jenkins(Github plugin) -> Jenkins hook url :http://<your host>:port/github-webhook/

In my case it is 
	http://152.14.93.228:8181/github-webhook
## Set up Slave Build Server
follow the rule mentioned in the first part and configure the Vagrantfile to disable the public network access

### Set the SSH connection between Slave and Master

1. create a pair of key on the master machine. Copy the public key to the slave machine

2. config the credentials on the slave machine to be the private key

3. check the ssh is successful between master and slave

## Set up Slave Node in Jenkins

Go to Manage Jenkins -> select Manage Nodes -> New Node (select Dumb Node) -> go to slave configure and set up the right credentials for master to set up ssh connections.

