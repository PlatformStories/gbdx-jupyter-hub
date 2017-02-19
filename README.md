# gbdx-jupyter-hub

This repo contains instructions and supplementary material in order to create a multi-user hub
which spawns, manages, and proxies multiple instances of the single-user Jupyter notebook server on
an AWS instance, using either GitHub or GBDX OAuth for authentication.

The implementation is based on [jupyterhub/jupyterhub-deploy-docker](https://github.com/jupyterhub/jupyterhub-deploy-docker) by the [Jupyter Project](http://jupyter.org/).

gbdx-jupyter-hub provides a demonstration environment in support of GBDX stories. It is not a production environment.


## Launch AWS instance

### AMI

64-bit Ubuntu

### Instance type

The recommendation [here](https://github.com/jupyterhub/jupyterhub/wiki/Deploying-JupyterHub-on-AWS) is a for a c4.2xlarge instance. According to the discussion [here](https://github.com/jupyterhub/jupyterhub/issues/505), you should budget for

* CPU: 0.5-1 / user
* memory: 0.5-1GB / user

CPU and memory are for concurrent users. If these numbers are true, then a c4.2xlarge instance should support about 15 concurrent users. We need to run tests and confirm. The on-demand cost of a c4.2xlarge instance is about $300/month.

### Storage

(Total number of users) x (GB / user)

Tentative number is 5GB / user.

### Security Group

Type | Protocol | Port | Source  
-----|-----|-----|-----|-----
SSH |   TCP | 22 | Custom 0.0.0.0/0
HTTPS | TCP | 443 | Custom 0.0.0.0/0
Custom TCP Rule | TCP | 8888 | Custom 0.0.0.0/0

### Keys

Pick a key so that communication with the hub is secure.


## Prerequisites

* Log into the instance and become root:

  ```bash
  ssh -i ~/keys/mykey.pem ubuntu@ec2-54-211-27-166.compute-1.amazonaws.com
  sudo -s
  ```

* Install [Docker](https://docs.docker.com/engine/installation/linux/ubuntulinux/).
To start Docker:

  ```bash
  service docker start
  ```

* Install [Docker Compose](https://docs.docker.com/compose/install/):

  ```bash
  curl -L https://github.com/docker/compose/releases/download/1.9.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
  chmod +x /usr/local/bin/docker-compose
  ```

* Create a self-signed certificate (entering information is optional) in `secrets` directory:

  ```bash
  mkdir secrets
  cd secrets
  openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout jupyterhub.key -out jupyterhub.crt
  cd ..
  ```

(Talk to justin.vetter@digitalglobe.com if you want to get an official domain name and SSL certificate.)

* Set up GitHub Authentication (you can skip this if you want to use GBDX Authentication exclusively). This requires creating a [GitHub application](https://github.com/settings/applications/new).
The fields should be filled in as follows:

  * Application name: GBDX Jupyter Hub (or any other name)
  * Homepage URL: https://ec2-54-211-27-166.compute-1.amazonaws.com/
  * Application description: GBDX Jupyter Hub
  * Authorization callback URL: https://ec2-54-211-27-166.compute-1.amazonaws.com/hub/oauth_callback
  * Optional: Use ```gbdx-jupyter-hub/digitalglobe-developers-logo.png``` for the application logo.  

Note down the Client ID and Client Secret.

* Set up GBDX Authentication (you can skip this if you want to use GitHub Authentication exclusively). Contact kostas.stamatiou@digitalglobe.com for admin access to OAuth.
The fields should be filled in as follows:

  * Name: GBDX Jupyter Hub
  * Domain: digitalglobe-platform.auth0.com
  * Allowed Callback URLs: https:///ec2-54-211-27-166.compute-1.amazonaws.com/hub/oauth_callback
  * Allowed Logout URLs: https:///ec2-54-211-27-166.compute-1.amazonaws.com/hub/oauth_callback

Note down the Client ID and Client Secret.


* Clone this repository:

  ```bash
  git clone https://github.com/digitalglobe/gbdx-jupyter-hub
  ```

* Clone the jupyterhub-deploy-docker repository:

  ```bash
  git clone https://github.com/jupyterhub/jupyterhub-deploy-docker
  ```

* Install make:

  ```bash
  apt-get update
  apt-get install make
  ```

## Build the user notebook server image

The user notebook server Docker image is found [here](https://github.com/PlatformStories/gbdx-notebook).

Clone the repository and build the image as follows:

```bash
git clone https://github.com/PlatformStories/gbdx-notebook
cd gbdx-notebook
docker build -t gbdx-notebook .
```

## Build the hub Docker image

* Move `secrets` into `jupyterhub-deploy-docker`:

  ```bash
  mv secrets jupyterhub-deploy-docker/
  ```

* Create a `userlist` file with authorized users in `jupyterhub-deploy-docker`. At a minimum, this file should contain a single admin username:

  ```bash
  admin_user_name admin
  ```

* Copy the files `Dockerfile.jupyterhub, jupyterhub_config.py, docker-compose.yml, .env` from `gbdx-jupyter-hub` to `jupyterhub-deploy-docker`
and make `jupyterhub-deploy-docker` your working directory.

  ```bash
  cd ~/gbdx-jupyter-hub
  cp Dockerfile.jupyterhub jupyterhub_config.py docker-compose.yml .env ~/jupyterhub-deploy-docker/
  cd ~/jupyterhub-deploy-docker
  ```

  * `Dockerfile.jupyterhub` is the Dockerfile for building the image of the hub. As of now (12/4/2016), we need to install [oauthenticator](https://github.com/jupyterhub/oauthenticator) and [dockerspawner](https://github.com/jupyterhub/dockerspawner) directly from the master branch because the latest changes have not been pushed to pypi.

  * `jupyterhub_config.py` is a configuration file for the hub container. You can authenticate using either Github or GBDX OAuth (and comment out the other). The following line ensures that a Docker volume can be created when the user name
  contains special characters.

    ```bash
    c.DockerSpawner.format_volume_name = dockerspawner.volumenamingstrategy.escaped_format_volume_name
    ```

  * `docker-compose.yml` is a configuration file for docker-compose. You can authenticate using either Github or GBDX OAuth (and comment out the other).

  * `.env` contains the values of environment variables used in `docker-compose.yml`. You need to specify the name of the user Docker image (default is protogen-notebook), the Github or GBDX OAuth client id and secret, depending on the chosen authentication method, and the oath callback url.

* Build the hub Docker image

  ```bash
  make build
  ```

## Run the Hub

Run the hub container in detached mode:

```bash
docker-compose up -d
```

Once the container is running, the JupyterHub console should be accessible at:

```
https://<ec2-public-ip>.compute-1.amazonaws.com
```

To stop the container:

```bash
docker-compose down
```

Note that stopping the hub container deletes the user containers. However, it does not delete the
corresponding data volumes. You can list these volumes with

```bash
docker volume ls
```

`jupyterhub-data` is the hub data volume which contains hub data like user cookies to persist across hub restarts.
If you delete it, you will have to remake the hub Docker image. You can delete a volume as follows:

```bash
docker volume rm <volume-name>
```

Finally, you can view the logs of a particular container as follows:

```bash
docker logs <container-name>
```

This can help with debugging.


## Some things to know

* The user data is stored in a volume. If you stop and restart the hub, the volume persists.
This is true even if you change the docker image.  
