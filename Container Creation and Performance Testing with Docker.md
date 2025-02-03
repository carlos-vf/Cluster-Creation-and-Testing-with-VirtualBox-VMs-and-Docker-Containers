# Container Creation and Performance Testing with Docker

## Goal
The goal is to set up and manage a multi-container application using Docker and Docker.

## Prerequisites
- Windows Subsystem for Linux (WSL 2).

## Docker Installation
Before starting, let's make sure that the version of the subsystem in the machine is WSL 2 (necessary to run Docker). You can run in your Windows PowerShell the following command
```
wsl -l -v
```
which should output your WSL version and distribution.
```
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

If your WSL version is 1, update it by running:
```
wsl --set-version Ubuntu 2
```

Update your package lists and install dependencies:
```
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

And sdd Docker’s official GPG key:
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Now let's proceed to set up the repository and install docker engine anc CLI:
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

We can check if docker is correctly installed by running:
```
docker --version
```

Finally, let's activate the service and run a test container:
```
sudo service docker start
docker run hello-world
```

## Dockerfiles



## Docker Compose

### Master Container

### Worker Containers

### Network



## Running the Containers

```
docker compose -f docker-compose.yml up --build -d
```

To enter the master node:
```
docker exec -it master sh
```



docker-compose down --volumes --remove-orphans
docker-compose up --build -d

Prune all the system:
```
docker-compose down
docker network prune -f
docker volume prune -f
docker system prune -af
```


Volume creation
```
sudo mkdir -p /mnt/volumes
sudo fallocate -l 500M /mnt/volumes/shared_volume.img  # Create a 500MB file
sudo mkfs.ext4 /mnt/volumes/shared_volume.img  # Format as ext4
sudo mkdir -p /mnt/shared_volume
sudo mount -o loop /mnt/volumes/shared_volume.img /mnt/shared_volume  # Mount it
```

docker ps
docker logs master

<p align="center">
  <img src="https://github.com/user-attachments/assets/80c94c17-1468-4f93-bc99-4459c112d987"  width="700">
</p>



