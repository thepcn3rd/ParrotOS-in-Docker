# ParrotOS-in-Docker

### Intoduction

The purpose of this repository is to share my notes on running ParrotOS in docker.  The main reason for building Parrot OS in this way was to escape the overhead of a hypervisor and to experiment with docker to learn it more.

### Objectives
1. Use Parrot OS Security and become familiar with the distro
2. Use Parrot OS to work on TryHackMe, HacktheBox, etc.
3. Use openvpn inside the docker container
4. Use X Forwarding or discover something better to use Firefox, Burp and other GUI based applications inside the docker
5. Learn Docker


### Installation of Docker on an Ubuntu Host
Using Ubuntu 22.04 with docker installed.  Nothing unique about docker other than not using the default ubuntu install.  

URL: [Install instructions for Ubuntu from Docker Site](https://docs.docker.com/engine/install/ubuntu/)

The uninstall of old versions was not exactly as shown, here is the command I used...
```bash
sudo apt-get remove docker docker.io containerd runc
```

Then follow the steps for "Set up the repository" and step 1 of "Install Docker Engine".  After the installation you should be able to run sudo docker run hello-world and it will test everything and return a message that it is working as expected.


### Setup of User Account to Execute Docker
This is an optional task but I like running docker commands as the least-privilege user that I am logged in with, without using sudo.  Depends on how you evaluate, this could be less secure but I will leave that assessment up to you.

To set this up you need to add the logged in user to the docker group that was recently created.  The below command will accomplish that... (I am using an example username of bob, bob would be who you log in with to Ubuntu as your default user.)

```bash
sudo usermod -aG docker thepcn3rd
```

After this configuration you will need to reboot for the user "bob" to be able to run docker commands without sudo.


### Setup ParrotOS in Docker
ParrotOS provides a docker image that can be pulled down with the following command.  I am not going to preface the commands with sudo due to configuring my user account.

```bash
docker pull parrotsec/security
```

After the download of the default image, run the following command to gain an interactive shell with the image running in a container.  I will not explain all of the command line switches.  The "--name" names the container that is running the image.  The "-v" creates a volume from a directory called work on my host and mounts it to the directory work in the container, this volume will contain files that I need to be keep saved.

```bash
mkdir ~/parrotWork
docker run --rm -ti --name parrot -v $PWD/parrotWork:/work parrotsec/security
```

After gaining an interactive shell it should look something like the following.
```shell_appearance
root@afc9a2444258c
#
```

To install the packages that you need to meet the objectives above run the following commands in the interactive shell in docker.  The below commands take about 10+ minutes to execute.  Execute after running "apt update"
```bash
apt install powershell netcat openvpn curl openssl ca-certificates fontconfig libxext6 libxrender1 libxtst6 parrot-interface-common ssh openjdk-17-jdk jupyter-notebook rlwrap default-mysql-client exiftool ffuf ldap-utils python3-pip smbclient remmina
```

The next commands will establish the ssh key between my Ubuntu host and the container.  (I would like to avoid using SSH, however I needed something to do X-Forwarding between the container and host.)  These commands are executed in the container.

```bash
ssh-keygen # If you want to establish the .ssh folder automagically
echo "public key from Ubuntu user account" > /root/.ssh/authorized_keys
chmod 400 /root/.ssh/authorized_keys
```

### Setup Work Folder for Burp and HacktheBox
In the work folder on the host from the host download burpsuite and place the hackthebox ovpn file inside this file.  To download burpsuite as a stand-alone java file run the following curl command

```bash
curl https://portswigger.net/burp/releases/download -o ~/parrotWork/burpsuite/burpsuite_community.jar
```


### Commit Changes and Create Docker Image

Now to commit the changes you made to the image run the following command from another Ubuntu shell.  Do not close or exit from the interactive shell to the docker container.

```bash
docker commit parrot parrot_v1
```

After running the above commit, I can execute "docker images" and observe a new image called parrot_v1 that I can use.

### Dedicated Network and Static IP for the Parrot Container

The docker container launches with a random IP Address being 172.17.0.x.  I wanted the address to be more static so I created a docker network on the host for ParrotOS by the following command

```bash
docker network create --subnet=172.31.1.0/24 parrotNetwork
```


### Post Commit of Creating the Docker Image
After I created the new image, exit the docker container by typing "exit" and hitting enter/return.  To launch the newly configured image run the following command.  (I placed this command in a launchParrot.sh bash script for future execution.)

I have also included the parrotNetwork and assigned the container when it is launched a static IP Address.

```bash
docker run --privileged --sysctl net.ipv6.conf.all.disable_ipv6=0 --rm -ti --name parrot -v $PWD/parrotWork:/work --net parrotNetwork --ip 172.31.1.10 parrot_v1
```

To explain the above command, you are launching the docker container to be privileged and disabling IPv6 due to compatiability with docker.  Again consider security when doing this.  This is your primary interactive container, if you close this window your container will shutdown.

By executing "docker ps" you should see your running docker container with the ParrotOS image that we created.  


### Launch SSH and openVPN from Container
From the interactive shell in the container you need to start ssh and then can launch the openvpn connection to hackthebox.

```bash
/etc/init.d/ssh start
openvpn lab_username.ovpn
```

The interactive window with then show the VPN is connected and this window should be left open and alone...

### SSH Interaction with the Container
Now to interact with the container and launch firefox, burp or other GUI applications you can use SSH with X-window forwarding.  I wrote the following script called sshParrot.sh to launch these applications.  You can also run the commands individually.

Part of the configuration I setup firefox to have profiles.  A profile that is default and then a profile for using the Burp or other proxy that is available.

For automation I use jupyter-notebooks frequently so I placed that in the script below.

```bash
#!/bin/bash

ip=172.31.1.10

ssh -X -fT root@$ip 'firefox -P default-esr &'
ssh -X -fT root@$ip 'firefox -P Proxy &'
ssh -X -fT root@$ip 'java -jar /work/burpsuite/burpsuite_community.jar &'

# Wait for firefox to launch
ssh -X -fT root@$ip 'jupyter-notebook --notebook-dir=/work/jnotebook --allow-root &'
```

The "ip" in the script is the initial IP assigned by docker, this may change as you have more containers executing.  Then I launch firefox and burpsuite and they automagically appear as windows on my host.

### docker exec Interaction with the Container
I would recommend not using ssh to interact with container as a shell.  I would recommend using the following command from the host to gain a bash interactive shell without the overhead of SSH.

```bash
docker exec -it parrot bash
```

### Run as a non-privileged user

Setup a user account in the docker that matches a user account on your host...

```bash
useradd -m -d /home/thepcn3rd -u 1000 -s /bin/bash thepcn3rd

# Set password for the user account created
passwd thepcn3rd
```

Then setup the user account to have sudo privileges...

```bash
usermod -aG sudo thepcn3rd
```

You will need to re-establish the ssh keys for the new user account and then modify the initial command to launch docker to the following (Does not allow sudo)...
```bash
docker run --privileged --sysctl net.ipv6.conf.all.disable_ipv6=0 --rm -ti --name parrot -v $PWD/parrotWork:/work --net parrotNetwork --ip 172.31.1.10 --user 1000:1000 parrot_v1
```


Last Updated: July 29, 2022

