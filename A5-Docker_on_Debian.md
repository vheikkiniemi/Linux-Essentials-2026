> [!NOTE]
> **🇬🇧 Note on the use of AI:** AI tools such as ChatGPT, Gemini, and Copilot were used to help develop this material.  
> **🇫🇮 Huomio tekoälyn käytöstä:** Tämän materiaalin laatimisessa on hyödynnetty tekoälytyökaluja, kuten ChatGPT:tä, Geminiä ja Copilotia.  

# A5 – Docker on Debian

## Part 1 – Run and Manage Your First Containers

Install Docker Engine on your Debian Server VM. Then use your regular user account to run, inspect, stop, and remove a Debian container and an Alpine container.

### 1. Install Docker Engine and configure user access

Follow the [official Docker Engine installation guide for Debian](https://docs.docker.com/engine/install/debian/#install-using-the-apt-repository). Use the `apt` repository method. After installing the packages, add your regular user account to the `docker` group before running your first container. [docs.docker.com](https://docs.docker.com/engine/install/debian/?utm_source=chatgpt.com)

```bash
sudo groupadd -f docker
sudo usermod -aG docker "$USER"
```

Log out and log back in so that your new group membership takes effect. If you connected over SSH, close the session and reconnect. Membership in the `docker` group grants root-level privileges on the Docker host. [Docker Docs](https://docs.docker.com/engine/install/linux-postinstall/)

Check your group membership and test Docker without `sudo`.

```bash
id -nG
```
```bash
docker run --rm hello-world
```

### 2. Run Debian and Alpine containers

Start both containers in the background using the standard images. Check that `docker ps` shows both containers with the status `Up`. [docs.docker.com](https://docs.docker.com/reference/cli/docker/container/run/)

```bash
docker run -dit --name debian-lab debian sh
```
```bash
docker run -dit --name alpine-lab alpine sh
```
```bash
docker ps
```

### 3. Use a shell inside each container

Open a shell in each container and read `/etc/os-release` to identify its Linux distribution. After checking each container, type `exit` to return to the Debian host terminal. [docs.docker.com](https://docs.docker.com/get-started/docker_cheatsheet.pdf)

```bash
docker exec -it debian-lab sh
```
```bash
cat /etc/os-release
```
```bash
exit
```
---

```bash
docker exec -it alpine-lab sh
```
```bash
cat /etc/os-release
```
```bash
exit
```

### 4. Stop and remove both containers

Stop both containers and verify that `docker ps -a` shows them with the status `Exited`. Then remove them and verify that both names are gone from the list. [docs.docker.com](https://docs.docker.com/reference/cli/docker/container/stop/)

```bash
docker stop debian-lab
```
```bash
docker stop alpine-lab
```
```bash
docker ps -a
```
---

```bash
docker rm debian-lab
```
```bash
docker rm alpine-lab
```
```bash
docker ps -a
```

### Part 1 – Summary

You installed Docker Engine on Debian, configured your user account to run Docker commands without `sudo`, and used the Debian and Alpine images to practice the container lifecycle. You started both containers, opened a shell in each, then stopped and removed them.

- **Pros:** Existing images let you start a Linux environment with a single command. You can inspect each container and manage its lifecycle independently.
- **Cons:** You need to record the commands and options to recreate the same setup later. Removing a container also removes changes stored in its writable layer. Membership in the `docker` group grants root-level privileges on the host.

---

## Part 2 – Build and Run Debian and Alpine Images (1 point)

Build two custom images, one based on Alpine and one based on Debian. Find both images in Docker’s local image list, run a script manually inside a container from each image, and then run the same script automatically using the Dockerfile’s default command.

### 1. Create a folder for each image

On your Debian Docker host, create the following folders and files.

```text
docker-part2/
├── alpine/
│   ├── Dockerfile
│   └── hello.sh
└── debian/
    ├── Dockerfile
    └── hello.sh
```

Save this script as `hello.sh` in **both** folders. Replace `YOUR NAME` with your own name or personal identifier. The script reads `/etc/os-release` to identify the distribution where it runs.

```sh
#!/bin/sh
. /etc/os-release
echo "Hello from YOUR NAME!"
echo "This script is running inside a $NAME container."
```

**`alpine/Dockerfile`**

```dockerfile
FROM alpine
COPY hello.sh /hello.sh
CMD ["sh", "/hello.sh"]
```

**`debian/Dockerfile`**

```dockerfile
FROM debian
COPY hello.sh /hello.sh
CMD ["sh", "/hello.sh"]
```

`FROM` selects the base image. `COPY` adds the script from the build context to the image. `CMD` sets the default command that runs when a container starts.

### 2. Build and find both images

Run these commands from the `docker-part2` directory. Each build uses its own folder as the build context.

```bash
docker build -t linux-hello-alpine ./alpine
```
```bash
docker build -t linux-hello-debian ./debian
```
```bash
docker image ls
```

Find `linux-hello-alpine` and `linux-hello-debian` in the `REPOSITORY` column.

### 3. Run the script manually inside each container

Start a named container from each image. The final `sh` in each `docker run` command overrides `CMD` and opens an interactive shell. Run `sh /hello.sh` **inside** the container, then type `exit` to return to the host. Exiting that shell stops the container.

```bash
docker run -it --name hello-alpine linux-hello-alpine sh
```
```bash
sh /hello.sh
```
```bash
exit
```
---

```bash
docker run -it --name hello-debian linux-hello-debian sh
```
```bash
sh /hello.sh
```
```bash
exit
```

Check that each greeting shows your personal identifier and the expected distribution.

### 4. Run the script automatically

Run each image again and let Docker use the `CMD` from its Dockerfile. The script prints the greeting and exits. The `--rm` option removes each of these new containers automatically when it exits.

```bash
docker run --rm linux-hello-alpine
```
```bash
docker run --rm linux-hello-debian
```

### 5. Remove the containers and custom images

Use `docker ps -a` to find the two stopped containers from step 3. Remove them first, then remove the two custom images.

```bash
docker ps -a
```
```bash
docker rm hello-debian
```
```bash
docker image rm linux-hello-debian
```
---

```bash
docker rm hello-alpine
```
```bash
docker image rm linux-hello-alpine
```

Verify that the named containers and custom images are gone from their respective lists.

```bash
docker ps -a
```
```bash
docker image ls
```

### Part 2 – Summary

You wrote separate Dockerfiles for Debian and Alpine, built both images, and found them in the local image list. You ran the greeting script manually from an interactive shell and automatically through the Dockerfile’s `CMD`. You then removed the containers and custom images.

- **Pros:** A Dockerfile records how to build an image. Every container created from that image starts with the same bundled script and default command.
- **Cons:** Changes to the bundled script require rebuilding the image and creating a new container. An unpinned base image tag can point to a different base image on a later build.

---

## Part 3 – Connect to Debian and Alpine Containers with SSH (1 point)

Build two images that run OpenSSH Server, one based on Debian and one based on Alpine. Start a container from each image, connect to both over SSH as a regular user, and verify that the user can run commands with `sudo`.

### 1. Create a folder for each image

On your Debian Docker host, create a new `docker-part3` directory with this structure.

```text
docker-part3/
├── alpine/
│   └── Dockerfile
└── debian/
    └── Dockerfile
```

Both Dockerfiles create a user named `student` with the password `ssh-lab-demo`. Use this password only for this exercise. Anyone who can read the Dockerfiles can see it.

### 2. Create the Dockerfiles

**`debian/Dockerfile`**

```dockerfile
FROM debian

RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-server sudo \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -s /bin/sh student \
    && echo 'student:ssh-lab-demo' | chpasswd \
    && echo 'student ALL=(ALL) ALL' > /etc/sudoers.d/student \
    && chmod 0440 /etc/sudoers.d/student

EXPOSE 22

CMD ["bash", "-c", "mkdir -p /run/sshd && ssh-keygen -A && exec /usr/sbin/sshd -D -e -o PasswordAuthentication=yes -o PermitRootLogin=no"]
```

**`alpine/Dockerfile`**

Alpine uses `apk` to install packages and `adduser` to create the account. Its OpenSSH server package is also named `openssh-server`. [Alpine Linux packages](https://pkgs.alpinelinux.org/package/edge/main/x86/openssh-server?utm_source=chatgpt.com)

```dockerfile
FROM alpine

RUN apk add --no-cache openssh-server openssh-keygen sudo \
    && adduser -D -s /bin/sh student \
    && echo 'student:ssh-lab-demo' | chpasswd \
    && echo 'student ALL=(ALL) ALL' > /etc/sudoers.d/student \
    && chmod 0440 /etc/sudoers.d/student

EXPOSE 22

CMD ["sh", "-c", "ssh-keygen -A && exec /usr/sbin/sshd -D -e -o PasswordAuthentication=yes -o PermitRootLogin=no"]
```

When each container starts, `ssh-keygen -A` generates any missing SSH host keys. The SSH server then runs in the foreground as the container’s main process. The `student` account can sign in with its password and use the same password when `sudo` prompts for it.

### 3. Build the images and start the containers

Run these commands from `docker-part3` on your Debian Docker host.

```bash
docker build -t linux-ssh-debian ./debian
```
```bash
docker build -t linux-ssh-alpine ./alpine
```
```bash
docker run -d --name ssh-debian -p 2222:22 linux-ssh-debian
```
```bash
docker run -d --name ssh-alpine -p 2223:22 linux-ssh-alpine
```
```bash
docker ps
```

Both containers listen on port `22` internally. Docker maps them to different ports on the host. [Docker Docs](https://docs.docker.com/engine/network/port-publishing)

| Container | Port on the Debian host | Port inside the container |
|---|---:|---:|
| `ssh-debian` | `2222` | `22` |
| `ssh-alpine` | `2223` | `22` |

### 4. Connect to both containers over SSH

From the Debian Docker host, connect to the Debian container. Enter `ssh-lab-demo` when prompted.

```bash
ssh -p 2222 student@<HOST-IP-ADDRESS>
```

Inside the SSH session, check your username, identify the Linux distribution, and verify that `sudo` works. `whoami` should print `student`, and `sudo whoami` should print `root`.

```sh
whoami
```
```bash
cat /etc/os-release
```
```bash
sudo whoami
```
```bash
exit
```

Then connect to the Alpine container and repeat the same checks. This time, `/etc/os-release` should identify Alpine Linux.

```bash
ssh -p 2223 student@<HOST-IP-ADDRESS>
```

### 5. Stop and remove both SSH containers and images

Exit the SSH session to return to the Debian Docker host. Stop both containers and check that `docker ps -a` lists them with the status `Exited`.

```bash
docker stop ssh-debian
```
```bash
docker stop ssh-alpine
```
```bash
docker ps -a
```

Remove both containers and images and verify that `ssh-debian` and `ssh-alpine` are gone from the list.

Containers:
```bash
docker rm ssh-debian
```
```bash
docker rm ssh-alpine
```
```bash
docker ps -a
```
Images:

```bash
docker image rm linux-ssh-debian
```
```bash
docker image rm linux-ssh-alpine
```
```bash
docker image ls
```

### Part 3 – Summary

You built Debian and Alpine images with OpenSSH Server, a regular user account, and `sudo`. You published each container’s SSH port on a different host port, signed in to both containers, verified `sudo` access, then stopped and removed the containers.

- **Pros:** SSH lets you use familiar Linux administration tools from an SSH client. The exercise also shows how two containers can both listen on port `22` while using different ports on the host.
- **Cons:** SSH adds a service and user accounts to maintain. A password written into a Dockerfile is visible to anyone who can read the file, so this setup is suitable for the lab only.

---

## Part 4 – Manage Debian and Alpine as Separate Docker Compose Projects (1 point)

Create two independent Docker Compose projects, one for Debian and one for Alpine. Build each image and start its container separately. Then verify each container’s hostname and environment variable before stopping and removing its container and image.

### 1. Create a new folder for each project

Create `docker-part4` alongside `docker-part3`. Copy the Debian and Alpine Dockerfiles from Part 3 into their respective folders. Create a separate `compose.yaml` next to each Dockerfile.

```text
docker-part4/
├── debian/
│   ├── Dockerfile
│   └── compose.yaml
└── alpine/
    ├── Dockerfile
    └── compose.yaml
```

**`debian/compose.yaml`**

```yaml
name: debian-ssh-lab

services:
  ssh:
    build: .
    image: compose-ssh-debian
    container_name: debian-ssh-lab
    hostname: debian-ssh-lab
    environment:
      LAB_ROLE: "Debian SSH server"
    ports:
      - "2222:22"
```

**`alpine/compose.yaml`**

```yaml
name: alpine-ssh-lab

services:
  ssh:
    build: .
    image: compose-ssh-alpine
    container_name: alpine-ssh-lab
    hostname: alpine-ssh-lab
    environment:
      LAB_ROLE: "Alpine SSH server"
    ports:
      - "2223:22"
```

Each `build: .` uses the Dockerfile in the same folder as its `compose.yaml`. Both projects use the service name `ssh`, and their distinct project names and host ports let them run independently. [docs.docker.com](https://docs.docker.com/reference/compose-file/build/)

### 2. Build and start the Debian project

From `docker-part4`, enter the `debian` folder. Build the image first, then start the container and inspect its settings.

```bash
cd debian
```
```bash
docker compose build
```
```bash
docker images
```
```bash
docker compose up -d
```
```bash
docker ps
```
```bash
docker exec -it debian-ssh-lab bash
```
```bash
hostname
```
```bash
env
```
```bash
exit
```

The container should report `debian-ssh-lab` as its hostname and `Debian SSH server` as its `LAB_ROLE`.

From the Debian Docker host, connect to the Debian container. Enter `ssh-lab-demo` when prompted.

```bash
ssh -p 2222 student@<HOST-IP-ADDRESS>
```

### 3. Build and start the Alpine project

Switch to the `alpine` folder and repeat the process using its Compose file.

```bash
cd ../alpine
```
```bash
docker compose up -d --build
```
```bash
docker ps
```
```bash
docker images
```
```bash
docker exec -it alpine-ssh-lab sh
```
```bash
hostname
```
```bash
env
```
```bash
exit
```

The container should report `alpine-ssh-lab` as its hostname and `Alpine SSH server` as its `LAB_ROLE`. Find both custom images, `compose-ssh-debian` and `compose-ssh-alpine`, in the image list.

Then connect to the Alpine container and repeat the same checks.

```bash
ssh -p 2223 student@<HOST-IP-ADDRESS>
```


### 4. Stop and remove each project separately

While still in the `alpine` folder, stop the Alpine container. Check that it appears as `Exited`, then remove the container and its image.

```bash
docker compose stop
docker compose ps -a
```
Next deletes image (alse stop container if running)

```bash
docker compose down --rmi all
docker compose ps -a
```

Switch to the `debian` folder and repeat the cleanup for the Debian project. `docker compose down --rmi all` removes the image used by the service in the current project. [docs.docker.com](https://docs.docker.com/reference/cli/docker/compose/down/)

```bash
cd ../debian
docker compose stop
docker compose ps -a
docker compose down --rmi all
docker compose ps -a
docker image ls
```

Check that each project’s final `docker ps -a` lists no containers and that both custom images are absent from `docker image ls`.

### Part 4 – Summary

You created separate Docker Compose projects for Debian and Alpine. Each `compose.yaml` defined its build context, image name, hostname, environment variable, and port mapping. You built, started, checked, stopped, and removed each project independently.

- **Pros:** Each Compose file records how to build and run its service, so you can repeat the setup without retyping the original `docker run` options. Separate projects let you manage one container without interrupting the other.
- **Cons:** You need to manage each project from its own directory or explicitly select its Compose file. Settings shared by both projects, such as a port-mapping convention, must be maintained in two files.