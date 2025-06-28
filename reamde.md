

# Ansible Lab with Docker

- Create docker container which will be accessable from host via ssh, so that ansible can run playbooks


## Docker Containers

```
.
├── Dockerfile.rocky
├── Dockerfile.ubuntu
├── docker-compose.yaml
├── hosts
└── reamde.md

```

#### Dockerfile.ubuntu
```
# Dockerfile.ubuntu
FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y openssh-server sudo python3 python3-pip && \
    mkdir /run/sshd

RUN useradd -m ansible -s /bin/bash && \
    echo 'ansible:password' | chpasswd && \
    echo 'ansible ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```


#### Dockerfile.rocky
```
# Dockerfile.rocky
FROM rockylinux:9

RUN dnf install -y openssh-server sudo python3 python3-pip && \
    ssh-keygen -A && \
    mkdir /run/sshd

RUN useradd -m ansible -s /bin/bash && \
    echo 'ansible:password' | chpasswd && \
    echo 'ansible ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

#### Building Custom image with SSH enabled
```
$ docker build -t ansible-ubuntu -f Dockerfile.ubuntu .
$ docker build -t ansible-rocky -f Dockerfile.rocky .
```

#### Creating Containers
```
$ docker network create ansible-lab

$ docker run -d --name ubuntu-node --network ansible-lab -p 2222:22 ansible-ubuntu
$ docker run -d --name rocky-node --network ansible-lab -p 2223:22 ansible-rocky
```


#### SSH From Host => container

- username: ansible 				: Set in Dockerfile.ubuntu
- password: password 				: Set in Dockerfile.rocky

```
$ ssh ansible@localhost -p 2222  # For Ubuntu
$ ssh ansible@localhost -p 2223  # For Rocky
```


#### SSH From Container => container
- username: ansible 				: Set in Dockerfile.ubuntu
- password: password 				: Set in Dockerfile.rocky


```
Step-1: Go inside a container [ here in ubuntu ]

	$ ssh ansible@localhost -p 2222  		: Method-1: for ubuntu-node
	$ docker exec -it ubuntu-node bash 		: Method-2: for ubuntu-node


Step-2: Then SSH to another container [ here rockylinux ]

	$ ssh ansible@rocky-node      			: Method-2: for rocky-node
```


#### SSH With keys, without password

Step-1: Generate Custom SSH Key pairs

```
$ ssh-keygen -t ed25519 -C ansible
	-> 	~/.ssh/ansible 				=> Error
	-> 	/home/riajul/.ssh/ansible 		=> OK
```



Step-2: Copy custom ssh public key to containers:

```
$ ssh-copy-id -i /home/riajul/.ssh/ansible  -p 2222  ansible@localhost 		: ubuntu-node
$ ssh-copy-id -i /home/riajul/.ssh/ansible  -p 2223  ansible@localhost 		: rocky-node
```


Step-3: Check is key copied

```
$ ssh -p 2222 	ansible@localhost 			: SSH with password into ubuntu-node, then
	# cat ~/.ssh/authorized_keys  			: => See Id ansible custom public copied here
	# exit
```

Step-4: Now use SSH key to login
```
$ ssh -i ~/.ssh/ansible -p 2222  ansible@localhost 	: SSH with ssh-key into ubuntu-node
	# exit
```


Step-5: Create Host files like:
```
[ubuntu]
ubuntu-node ansible_host=localhost ansible_port=2222

[rocky]
rocky-node ansible_host=localhost ansible_port=2223

[all:vars]
ansible_user=ansible
ansible_password=password
ansible_connection=ssh
ansible_python_interpreter=/usr/bin/python3
```

Step-6: Test ansible script: Ad-hoc command

```
$ sudo apt install sshpass
$ ansible all -i hosts -m ping
```


Get IPs

docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ubuntu-node
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' rocky-node



## Docker compose Setup

```
.
├── Dockerfile.rocky 				: Same as above
├── Dockerfile.ubuntu 				: Same as above
├── docker-compose.yaml
├── hosts
└── reamde.md

```

#### docker-compose.yaml

Just make little changes

- instead of `obunto-node` it become `ubuntu_svc`

```
services:
  ubuntu_svc:
    build:
      context: .
      dockerfile: Dockerfile.ubuntu
    image: ansible_ubuntu
    container_name: ubuntu_container
    ports:
      - "2222:22"
    restart: always
    networks:
      - ansible_lab

  rocky_svc:
    build:
      context: .
      dockerfile: Dockerfile.rocky
    image: ansible_rocky
    container_name: rocky_container
    ports:
      - "2223:22"
    restart: always
    networks:
      - ansible_lab

networks:
  ansible_lab:
    driver: bridge
```


```
$ docker compose up -d 				: Run all services
$ docker compose config --services 		: See services name

$ docker compose down 				: To delete containers
```




#### SSH From Host => container

- username: ansible 				: Set in Dockerfile.ubuntu
- password: password 				: Set in Dockerfile.rocky

```
$ ssh -p 2222  ansible@localhost 		# For Ubuntu
$ ssh -p 2223  ansible@localhost   		# For Rocky
```


#### SSH From Container => container
- username: ansible 				: Set in Dockerfile.ubuntu
- password: password 				: Set in Dockerfile.rocky


```
Step-1: Go inside a container [ here in ubuntu ]

	$ ssh -p 2222 ansible@localhost  	: Method-1: for ubuntu-node
	$ docker exec -it ubuntu_svc bash 	: Method-2: for ubuntu-node


Step-2: Then SSH to another container [ here rockylinux ]

	$ ssh ansible@rocky_svc      		: By service name
	$ ssh ansible@rocky_container      	: By container name
```

####  Inventory file

Inventory file or host file can be any name, but common are

- inventory.ini
- hosts.ini
- hosts

```
[ubuntu]
# localhost:2222 					# Short version
ubuntu_svc ansible_host=localhost ansible_port=2222 	# Long Version

[rocky]
rocky_svc ansible_host=localhost ansible_port=2223

[all:vars]
ansible_user=ansible
ansible_password=password
ansible_connection=ssh
ansible_python_interpreter=/usr/bin/python3
```


```
$  ansible all -i hosts -m ping
```
