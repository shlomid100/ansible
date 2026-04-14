# ansible learning
----------------------

#Dockerfile.slave
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y openssh-server python3 && \
    mkdir /var/run/sshd
#set root password (simple for lab)
RUN echo 'root:root' | chpasswd
#allow root login
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]

#Dockerfile.master
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y ansible openssh-client python3 sshpass iputils-ping
WORKDIR /ansible
CMD ["bash"]

# In Git Bash create the two images
docker build -t shlomid100/ansible-slave -f Dockerfile.slave .
docker build -t shlomid100/ansible-master -f Dockerfile.master .

# Create the bridge to connect master & slave
docker network create -d bridge myansiblenet

# Create the Containers of master & slave, we can first create the network and then in container command connect it to the network OR
# OR create the container and later connect it to the network, first in create connect to network
docker run -d --name slave1 --network myansiblenet -p 8080:80 shlomid100/ansible-slave #slaves → background so only -d
docker run -d --name slave2 --network myansiblenet -p 8080:80 shlomid100/ansible-slave #slaves → background so only -d
docker run -idt --name master --network myansiblenet shlomid100/ansible-master #master need -it → interactive (so you can run ansible)
# OR
docker run -d --name slave1  ansible-slave  --> Then run docker network connect myansiblenet slave1
docker run -d --name slave2  ansible-slave  --> Then run docker network connect myansiblenet slave2
docker run -it --name master ansible-master --> Then run docker network connect myansiblenet master

# Go inside the master
docker exec -it master bash // bash since it's ubuntu

# Ping to the slaves
ping slave1 OR ping slave2 //172.19.0.2, 172.19.0.3, master 172.19.0.4

# Take the IP from the container OR can be taken also from ping comand from inside the master.
docker inspect slave1 | grep IP

# We can do SSH from master to slaves and run on it commands, For first time need to do it to all slaves as it's asking for fingerpring and we set yes, if not when later we use the
# global ping to all slaves with inventory it will failed as it's not waiting for approve yes for fingerprint, user/pass for slave we set in Dockefile.slave in line: RUN echo 'root:root' | chpasswd
# we can overcome it by few ways: in docker run add -e ANSIBLE_HOST_KEY_CHECKING=False, in Dockerfile add ENV ANSIBLE_HOST_KEY_CHECKING=False. just run befor ping:
# export ANSIBLE_HOST_KEY_CHECKING=False, we can add host to known_hosts and thus no need permision like: 
ssh root@slave1, ssh root@slave2 


# Create the inventory file with all nodes details so we can control from master=control machine on groups of nodes, we don't have vim on master so use cat instaed of install
cat > inventory <<'EOF'
[slaves]
slave1 ansible_user=root ansible_password=root
slave2 ansible_user=root ansible_password=root
EOF

# Change access to the file
chmod 777 inventory

# Ping to all slaves in one command
ansible slaves -i inventory -m ping

# I added new slave3 and i will add it to the network and add it to the known_hosts
docker run -d --name slave3 --network myansiblenet shlomid100/ansible-slave

# Take it's IP
docker inspect slave3 | grep IP // 172.19.0.5

# If not connect the container to net do it manually
docker network connect myansiblenet slave3

# Add the slave3 to the known_hosts, Inside the master container run: it connects to slave2 then fetches its SSH fingerprint then appends it to to the file content.
ssh-keyscan slave2 >> /root/.ssh/known_hosts
#OR for all together
ssh-keyscan slave1 slave2 slave3 >> /root/.ssh/known_hosts

# Check it appendded 
cat /root/.ssh/known_hosts

# Add new slave to inventory file:
echo "slave3 ansible_user=root ansible_password=root" >> inventory

# Go inside the master and run ping with inventory to all slaves
docker exec -it master bash
ansible slaves -i inventory -m ping

# resoult:
slave1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
slave3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
slave2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

*********************************************************

#will delete the container and inventory file will be deleted so before delete copy the file to host machine c:/temp, from inside master: 
docker exec -t master bash
docker cp master:/ansible/inventory c:/temp

#create the container with hostname master & with volume so always we will have the files saved in host, we added MSYS_NO_PATHCONV=1 and set path in " as Bash not know to convert paths:
MSYS_NO_PATHCONV=1 docker run -idt --name master --hostname master --network myansiblenet -v "$(pwd):/ansible" shlomid100/ansible-master

PLAYBOOK:
I created inside container playbook.yml like below :
   - hosts: slaves
     become: yes --> installing packages requires root privileges, Even if you connect as root, it’s best practice (and avoids surprises).
     tasks:
       - name: install apache
         apt:
           name: httpd
           state: present
           update_cache: yes --> Run the equivalent of:apt update before installing the package. this work like this in ubuntu if not do apt update first the system might not know about new packages or may try to install                                   outdated versions or even fail with “package not found”
       - name: start apache
         service:
            name: apache2
            state: started
            enabled: yes

# Run the playbook to install the all slaves the apache2:
  ansible-playbook -i inventory playbook.yml
# Check if service up: 
  ansible -i inventory slaves -m shell -a "service apache2 status"

# Push the Images to DockerHub
docker push shlomid100/ansible-master
docker push shlomid100/ansible-slave














  
