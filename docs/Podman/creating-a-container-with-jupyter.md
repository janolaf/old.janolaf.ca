---
title: "Podman and Jupyter"
date: 2022-02-26T18:43:04-06:00
draft: false
toc: false
images:
categories:
  - toot
tags:
  - untagged
---
## Podman
Podman is a daemonless, rootless, container engine. Podman is designed to be a
drop-in replacement for Docker. You can run Docker Compose files with a little
extra setup. Some other advantages of podman is the ability to generate systemd
service files or kubernetes Pod yaml files. 

## Deploying Jupyter Notebook with podman

Starting with something simple, lets deploy Jupyter Notebooks.

Podman will automatically create the necessary volume for your image if you do
not specify a location.

Most guides recommend doing something like this:

```Dockerfile:
podman container run -d \
    --name jupyter \
    -p 6000:8888 \
    -v jupyter-work:/home/jovyan/work:Z \
    jupyter/scipy-notebook
```
This is okay, but then we have to get the token from the terminal. You run
'podman logs jupyter', copy the string after token=, it is all quite clunky
really. 

What we can do is create our own token or disable tokens completely by adding
"start-notebook.sh --NotebookApp.token=''"  after scipy-notebook. If you want
to create a token add --NotebookApp.token='my_token', replacing my_token with
whatever you like. Much better than a random string of gibberish.

If systemd is managing the container, and we will be generating a systemd
service file, we can auto update the container as well by adding a '--label
io.containers.autoupdate={local,registry}'. 

If you want to run JupyterLab, -e JUPYTER_ENABLE_LAB=yes.

Like Docker, if you need to access the container from a browser, you need to
tell Podman what port the container will listen on. The syntax is '-p
host:container'. In the example below, we are going to access Jupyter Notebooks
from port 6000. The container internal port is 8888.

If you are deploying on a system running SELinux, you need to add a z or Z to
the end of volumes. If the volume will be accessed by more than one container,
use z, otherwise use Z. If you want to create a container tailored SELinux
policy, look into udica.  

```bash:
podman container run -d \
    --name jupyter \
    --label io.containers.autoupdate=registry \
    -p 6000:8888 \
    -e JUPYTER_ENABLE_LAB=yes \
    -v /home/user/jupyter:/home/jovyan/work:Z \
    docker.io/jupyter/scipy-notebook start-notebook.sh --NotebookApp.token=''
```
With the above command we create a container named "jupyter", accessible from
localhost:6000. Files will be saved and accessible from the host at ~/jupyter.
Using Jupyter’s scipy-notebook image from Docker.io. Podman will check
docker.io for new images. Notebook token is disabled.

Check if the container is running properly
```Dockerfile:
podman container ps
```
If there's a problem, run 'podman logs jupyter'.

## Generating systemd service file

If everything is working as intended, all that is left is to generate the
systemd service file, enable it and see if it works.

If it does not already exist, create the systemd user directory in your home
directory.

```bash:
mkdir -p ~/.config/systemd/user
```

Then generate the systemd service file.

```bash:
podman generate systemd \
	--user \
	--new \
	--name jupyter \
	-f /home/user/.config/systemd/user/
``` 
This will create container-jupyter.service in ~/.config/systemd/user/. If you
do not use --name you will get a service file named
container-dg6464323fsdf34536...

After creating the service file, reload systemd.
```
Systemctl --user daemon-reload
```
Stop the jupyter container.
```bash:
podman stop jupyter
```
Enable and start container-jupyter.service.
```bash:
Systemctl --user enable --now container-jupyter.service
```
Check if everything worked
```bash:
Systemctl --user status container-jupyter
```
```bash:
podman container ps jupyter
```
If you want Jupyter notebook to be accessible when the user is not logged in,
you will need to run 

```bash:
loginctl enable-linger user
```
## Conclusion

That is it. Simple and easy.
Podman is great for developing and testing Kubernete Pods. Podman can convert
Docker or Docker-Compose file to kubernetes yaml.
