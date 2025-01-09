# docker
This is a help document to set up a server noting some (but not all important steps) 

## Initial set up
### Create user(s) and change initial passwords
1. create new user with automatic home directory using 
    ```bash 
    sudo useradd -m new_username
    ```
2. give user root privileges 
    ```bash
    sudo usermod -aG sudo new_username
    ``` 
3. check if it worked 
    ```bash 
    groups new_username
    ``` 
    output should include sudo
4. Setup user password with
    ```bash
    sudo passwd new_username
    ```
    and change root password with
    ```bash
    sudo passwd root
    ```
5. switch to new_user

*From here on, you can also disable password access all together and only use shh-keys (which you should)*

### ssh-keygen and access
You can google that

### Firewall setup
Set up UFW (uncomplicated firewall) with
```bash
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow 22/tcp # SSH
sudo ufw enable
```

## Docker
### Install docker
install docker following the instructions on the [website](https://docs.docker.com/engine/install/debian/) and add your user to the docker group 

```bash 
sudo usermod -aG sudo $USER
```
Log out and back in and test if docker works correctly
```bash
docker run hello-world
```
**Do not use ```sudo``` with docker, you don't want docker containers have root privileges!**

### Set up docker network with reverse proxy (nginx)
Clone this repo on your server.

Now start up the proxy manager
```bash
cd docker/npm
docker compose up -d 
```
this builds the docker container from the docker-compose.yaml and runs it in the background. 

*the reverse proxy will listen on the ports specified in the yaml-file, even though these ports are not actively allowed by ufw!*

You can now start stetting up nginx. Got to server.address:81 in the browser to get to the **NGINX Proxy Manager**. Log in with the standard credentials
```
Email:    admin@example.com
Password: changeme
```
**and change them immediately!**

### Hello-world test

Use 

```bash
cd docker/helloworld
docker compose up -d 
```
to start the hello world test homepage. The website will be mapped to port 8080. 

Got to the Nginx Proxy Manager create a new proxy host:

```bash
Domain Name: hello.server.name  # whichever subdomain you want to use
Scheme: http # for now until we know how to do https
Forword Hostname / IP: hello-world # this is the name of the application speficied in the yaml-file of helloworld
Forward Port: 8080 # because that's the port the docker container is mapped to by default
```
and save. 
If you now go to hello.server.name you should see the hello-world test homepage. Great, everything works. 

*Using the name given in the yaml-file at Forward Hostname works, because the nginx-container and the hello-world container are in the same docker network called npm. Hence nginx can find hello-world by name.*

### Jupyter-Notebook

We use Juypter-Notebook from [here](https://quay.io/repository/jupyter/base-notebook). With 
```bash
cd docker/jupyter
docker compose up -d 
```
we start the jupyter notebook container and we add a proxy host which forwards incoming traffic to port 8888, where the notebook app listens by default. Also we need to enable "Websockets Support" otherwise Jupyter will not be able to reach the Kernel.

To access the notebook we need a token by default, which we can find by checking which jupyter servers are running

```bash
docker exec -it container_id /bin/bash # opens an interactive terminal session in the jupyter container with bash
jupyter server list #lists all running jupyter notebook instances running in this container, i.e. one
>> Currently running servers:
>> http://container_id:8888/?token=some_digits_and_letters_making_up_the_token :: /home/jovyan
exit # terminates the interactive terminal session of the container
```
We can now go to the subdomain where we find jupyter notebook and use the token to get to the notebook

*there is also a way to use username and password, check the [docs](https://jupyter-docker-stacks.readthedocs.io/en/latest/index.html) for the how to*

## HTTPS
TODO