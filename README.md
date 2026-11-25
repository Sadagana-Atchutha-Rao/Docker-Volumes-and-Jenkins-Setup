# Docker-Volumes-Storage-Resource-Management-Jenkins-Setup

## 📌 Overview

In this module, I learned how Docker manages persistent data, shares storage between containers, mounts host directories, uses AWS EBS storage, and controls container resource utilization.

I also practiced Docker system commands, configured resource limits, and launched Jenkins using a Docker container.

## 📚 Topics Covered

* Docker Volumes and Persistent Storage
* Named and Anonymous Volumes
* Sharing Volumes Between Containers
* Creating Volumes Using a Dockerfile
* Creating and Inspecting Docker Volumes
* Volume Mounting
* Bind Mounts
* tmpfs Mounts
* Sharing Local Files with Containers
* Integrating AWS EBS Volumes with Docker
* Docker System Commands
* Docker Resource Management
* Jenkins Setup Using Docker
* Container and Image Cleanup

---

## 1. Docker Volumes

### What is a Docker Volume?

A Docker Volume is a storage mechanism used to persist data generated and used by containers.

Container filesystems are generally temporary. If a container is deleted, its writable-layer data is lost. Volumes allow application data to survive container deletion.

### Key Points

* Volumes store data outside the container's writable layer.
* Data persists when a container stops or is removed.
* Multiple containers can share the same volume.
* Volumes are useful for databases, application data, and Jenkins home directories.
* Docker manages named volumes.

### Docker Storage Options

| Storage Type     | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| Named Volume     | A Docker-managed volume with a user-defined name.            |
| Anonymous Volume | A Docker-managed volume created without a user-defined name. |
| Bind Mount       | Maps a specific host path to a container path.               |
| tmpfs Mount      | Stores temporary data in host memory (RAM).                  |

---

## 2. Method 1: Creating a Volume Using a Dockerfile

### Step 1: Create a Dockerfile

```dockerfile
FROM ubuntu
VOLUME ["/volume1"]
```

### Step 2: Build the Docker Image

```bash
docker build -t reyaz:v1 .
```

### Step 3: Create and Run a Container

```bash
docker run -it --name cont1 reyaz:v1
```

### Step 4: Create Files Inside the Volume

```bash
cd /volume1
touch file{1..5}
ls
```

### Step 5: Locate Docker Volume Data

On a typical rootful Linux Docker Engine installation, Docker-managed volume data is stored under:

```bash
/var/lib/docker/volumes/
```

List the volumes:

```bash
docker volume ls
```

Inspect the container's mounts:

```bash
docker inspect cont1
```

**Note:** The `VOLUME` instruction in this Dockerfile creates an anonymous volume when a container is created. Docker assigns its name automatically.

---

## 3. Sharing a Volume Between Containers

Docker allows multiple containers to access the same volume.

### Step 1: Create the First Container

```bash
docker run -it --name cont1 reyaz:v1
```

Create files inside `/volume1`.

```bash
cd /volume1
touch file{1..5}
```

Detach without stopping the container using `Ctrl+P`, followed by `Ctrl+Q`.

### Step 2: Create Another Container Using the Same Volume

```bash
docker run -it --name cont2 --volumes-from cont1 ubuntu
```

Check the shared files:

```bash
cd /volume1
ls
```

### Step 3: Create a Third Container

```bash
docker run -it --name cont3 --volumes-from cont2 ubuntu
```

The containers can access the same underlying volume.

**Important:** `--volumes-from` shares the source container's mounts. It does not copy the volume's data.

---

## 4. Method 2: Creating a Volume Directly with Docker

Create a container with an automatically created anonymous volume.

```bash
docker run -it --name cont4 -v /volume2 ubuntu
```

Create sample files:

```bash
cd /volume2
touch python{1..5}
```

Detach using `Ctrl+P`, followed by `Ctrl+Q`.

Create another container that shares the volume:

```bash
docker run -it --name cont5 --volumes-from cont4 ubuntu
```

Check the data:

```bash
cd /volume2
ls
```

Create additional files:

```bash
touch python{6..10}
```

The same volume can be accessed from `cont4` and `cont5`.

---

## 5. Method 3: Creating and Mounting a Named Volume

### Step 1: Create a Named Volume

```bash
docker volume ls
docker volume create volume3
```

### Step 2: Inspect the Volume

```bash
docker volume inspect volume3
```

The output includes the volume name and its mountpoint.

### Step 3: Mount the Volume in a Container

```bash
docker run -it --name cont6 \
  --mount source=volume3,target=/volume3 \
  ubuntu
```

### Step 4: Create Files

Inside the container:

```bash
cd /volume3
touch html{1..5}
ls
```

### Step 5: Share the Volume with Another Container

```bash
docker run -it --name cont7 \
  --mount source=volume3,target=/volume3 \
  ubuntu
```

Both containers access the same named volume.

### Important Commands

```bash
docker volume create volume3
docker volume ls
docker volume inspect volume3
docker volume rm volume3
```

A volume cannot be removed while it is still in use by a container.

---

## 6. Sharing Local Host Files with a Docker Volume

Create sample files on the Docker host:

```bash
touch reyaz{1..10}
```

Inspect the named volume:

```bash
docker volume inspect volume3
```

Copy the files into its `_data` directory using the mountpoint reported by Docker.

For example, on a standard rootful Linux Docker Engine:

```bash
sudo cp reyaz* /var/lib/docker/volumes/volume3/_data/
```

The files will be accessible inside containers that mount `volume3`.

**Best practice:** Use bind mounts when you want to map an existing host directory directly into a container. Avoid relying on Docker's internal volume directory as a general file-sharing interface.

---

## 7. Using AWS EBS Volumes with Docker Containers

Amazon Elastic Block Store (EBS) provides block storage for Amazon EC2 instances.

An EBS volume can store data used by Docker containers.

### Step 1: Create an EBS Volume

* Open the AWS EC2 console.
* Create a 20 GB EBS volume.
* Select the same Availability Zone as the target EC2 instance.
* Attach it to the instance.

### Step 2: Identify the Attached Device

```bash
lsblk
```

Identify the new disk carefully. On Nitro-based instances, an EBS volume may appear under a name such as `/dev/nvme1n1`.

### Step 3: Format the Disk

Only format a confirmed new, empty disk. Formatting an existing filesystem can destroy data.

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

### Step 4: Create a Mount Directory

```bash
sudo mkdir -p /mnt/ebs-volume
```

### Step 5: Mount the EBS Filesystem

```bash
sudo mount /dev/nvme1n1 /mnt/ebs-volume
```

Verify the mount:

```bash
df -h
lsblk
```

### Step 6: Mount EBS Storage into a Container

```bash
docker run -it --name my-container1 \
  -v /mnt/ebs-volume:/mynewebsvolume \
  ubuntu
```

Inside the container:

```bash
cd /mynewebsvolume
touch hello
ls
```

The file is stored on the EBS-backed host filesystem.

### Step 7: Share the Same Mount with Another Container

```bash
docker run -it --name mynewcont2 \
  --volumes-from my-container1 \
  ubuntu
```

Inside the second container:

```bash
cd /mynewebsvolume
ls
```

The second container can access the same files.

**Important:** `--volumes-from` shares the host bind mount from the first container. EBS data remains separate from the container's writable layer. Device names can vary, so always confirm the correct disk before formatting or mounting it.

---

## 8. Bind Mounts

### What is a Bind Mount?

A bind mount maps a specific directory or file on the Docker host to a path inside a container.

It is useful for development, static websites, configuration files, and source-code sharing.

### Step 1: Create a Website Directory

```bash
mkdir website
echo "Hello Docker" > website/index.html
```

### Step 2: Run an Nginx Container

```bash
docker run -d \
  --name mywebsite \
  -p 80:80 \
  -v "$(pwd)/website:/usr/share/nginx/html" \
  nginx
```

### Step 3: Access the Website

Open the EC2 instance's public IP address in a browser.

Ensure the EC2 security group allows inbound HTTP traffic on port 80.

### Step 4: Modify the Website

```bash
echo "Hello Mr Docker" > website/index.html
```

Refresh the browser to see the updated content.

The host directory and the container's mounted directory refer to the same underlying files.

### Advantages

* Immediate file sharing between host and container.
* Easy development and testing.
* Useful for website content and configuration files.
* Changes can be reflected without rebuilding the image.

---

## 9. tmpfs Mounts

### What is tmpfs?

A tmpfs mount stores temporary data in host memory rather than in a persistent disk-backed filesystem.

### Example

```bash
docker run -d \
  --name nginx-cache \
  --tmpfs /app/cache \
  nginx
```

### Key Characteristics

* Uses memory (RAM).
* Suitable for temporary caches and transient data.
* Does not provide persistent storage.
* Data is lost when the container stops or the tmpfs mount is destroyed.
* Can be useful for sensitive temporary data that should not be written to disk.

**Note:** tmpfs data may still be subject to host memory and swap behavior. Do not treat tmpfs alone as a complete security guarantee.

---

## 10. Docker Volume Interview Questions

### Q1. What is a Docker Volume?

A Docker Volume is Docker-managed storage that allows data to persist independently of a container's writable layer.

### Q2. What is the difference between a Named Volume and a Bind Mount?

**Named Volume:** Managed by Docker and identified by a name. Suitable for persistent application data.

**Bind Mount:** Maps a specific host path into a container. Suitable for source code, website files, and configuration.

### Q3. What is an Anonymous Volume?

An Anonymous Volume is a Docker-managed volume without a user-defined name. Docker assigns it an automatically generated identifier.

### Q4. What is a tmpfs Mount?

A tmpfs mount stores temporary data in memory instead of using a normal persistent disk-backed filesystem.

### Q5. Can multiple containers share a volume?

Yes. Multiple containers can mount the same volume, subject to filesystem behavior and application-level concurrency requirements.

### Q6. What happens to a named volume when a container is deleted?

The named volume normally remains available until it is explicitly removed.

### Q7. What is the difference between `-v` and `--mount`?

Both can configure mounts. `-v` is a concise syntax, while `--mount` provides an explicit key-value format and is generally easier to read for more complex configurations.

### Q8. Why are volumes useful for Jenkins?

Jenkins stores jobs, plugins, configuration, and build-related data in its home directory. Mounting persistent storage at `/var/jenkins_home` helps retain that data when the container is recreated.

---

## 11. Docker System Commands

### Check Docker Storage Usage

```bash
docker system df
```

Display detailed storage information:

```bash
docker system df -v
```

### List Containers

```bash
docker ps
docker ps -a
```

### Inspect a Container

```bash
docker inspect cont5
```

Filter mount-related details:

```bash
docker inspect cont5 --format '{{json .Mounts}}'
```

### Inspect Docker Volumes

```bash
docker volume ls
docker volume inspect volume3
```

These commands help troubleshoot storage, inspect mounts, and understand Docker resource usage.

---

## 12. Docker Resource Management

By default, containers do not have CPU and memory limits unless limits are configured.

Containers share the host's available CPU and memory resources. Resource limits help prevent a single container from consuming excessive resources.

### Check Live Resource Usage

```bash
docker stats
```

Check a specific container:

```bash
docker stats cont7
```

### Create a Container with CPU and Memory Limits

```bash
docker run -it \
  --name cont7 \
  --cpus="0.1" \
  --memory="300m" \
  ubuntu
```

* `--cpus="0.1"` limits the container to approximately 10% of one CPU's capacity.
* `--memory="300m"` sets a memory limit of approximately 300 MiB.

### Update Resource Limits

```bash
docker update cont7 \
  --cpus="0.7" \
  --memory="300m"
```

The CPU setting allows up to approximately 70% of one CPU's capacity.

### Important Resource Management Commands

```bash
docker stats
docker inspect cont7
docker update cont7 --cpus="0.5"
```

Resource limits help improve host stability and resource allocation.

---

## 13. Jenkins Setup Using Docker

Jenkins is a CI/CD automation server used to build, test, and deploy applications.

### Step 1: Run Jenkins in Docker

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  jenkins/jenkins:lts
```

This starts Jenkins in a container and publishes its web interface on port 8080.

### Step 2: Open a Shell Inside the Container

```bash
docker exec -it jenkins /bin/bash
```

### Step 3: Retrieve the Initial Administrator Password

```bash
docker exec jenkins \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

Open the Jenkins interface at:

`http://EC2-PUBLIC-IP:8080`

Configure the EC2 security group to permit port 8080 only from trusted IP addresses.

### Recommended: Persist Jenkins Data

For a more durable setup, create a named volume:

```bash
docker volume create jenkins_home
```

Run Jenkins with persistent storage:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

The volume helps retain Jenkins configuration, plugins, and job data when the container is replaced.

---

## 14. Deleting Containers and Images

**Warning:** These commands can permanently delete containers, application state, or images. Review the targets before running them.

### List All Containers

```bash
docker ps -a
```

### Stop and Remove All Containers

```bash
docker rm -f $(docker ps -aq)
```

If no containers exist, the command may report an error because the substitution is empty.

### List All Images

```bash
docker images
```

### Remove All Images

```bash
docker rmi -f $(docker images -q)
```

Removing images may fail if images are still referenced or used by containers.

**Best practice:** Avoid running bulk cleanup commands on production servers without checking which resources will be removed.

---

## 15. Summary

In this module, I practiced:

* Creating Docker volumes using Dockerfiles and CLI commands.
* Understanding named and anonymous volumes.
* Sharing storage between containers.
* Mounting Docker-managed volumes.
* Sharing local host files with containers.
* Integrating AWS EBS storage with Docker.
* Configuring bind mounts and tmpfs mounts.
* Inspecting Docker storage and container resources.
* Setting CPU and memory limits.
* Running Jenkins inside Docker with persistent storage.

### Key Takeaway

Docker volumes and mounts separate application data from the container's writable layer. They are essential for persistent storage, development workflows, shared data, and reliable containerized applications.
