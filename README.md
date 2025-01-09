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

