---
id: creating-a-container
title: "Containers with Podman" 
tags: [ podman, jupyter, systemd ]
---

Podman is a daemonless, rootless, container engine, which is designed to be a drop-in replacement for Docker. As a drop-in replacement for Docker, Podman supports the same commands and syntax. 

Podman can support Docker Compose files if you install `podman-compose`. Some of the other advantages of using podman, is the ability to generate systemd service files or kubernetes Pod yaml from docker or docker-compose files.

# Deploying a Container with Podman

To run a container

```bash
podman container run -d ngnix
```

`-d` or `--detach` means to run the contaier in the background.

## Ports

Like Docker, if you need to access the container from a browser, you need to tell Podman what port the container will listen on. You use `-p` or `--port` The syntax is `-p host:container`. 

for example if we want to access ngnix from 8080, we would add '-p 8080:80'

```bash
podman container run -d -p 8080:80 ngnix 
```

## Volumes

Use `-v host-directory:container-directroy` or `--volume host-directory:container-directroy`

If not instructed, podman will create a volume with a unique ID. You can point to a already created volume or host directory.

```bash
podman container run -d -v /srv/foo:/foo ngnix
```

You can also create a volume

```bash
podman volume create -name foo
```

have the container use the volume

```bash
podman container run -d -v foo:/foo ngnix
```

This will create a volume called `foo`, stored in `~/.local/share/containers/storage` if created by non-root user. `/var/lib/container/storage` if created by root. This can be changed in `/etc/container/storage.conf` or `~./config/container/storage.conf`

You can add the volume with `-v foo:/var`

:::info 
If you are deploying a rootless contaner on a system running SELinux, you need to add `:z` or `:Z` to the end of volumes. If the volume will be accessed by more than one container, use `z`. If the volume is only accessed by one container use `Z`. If you want to create a container tailored SELinux policy, look into [udica](https://github.com/containers/udica). 
:::

## Environment Variables

Use `--env` or `-e`

Envorimetnal variables is used to configure each container, like user name, passwords, ip of database.


## Generating systemd service file

If everything is working as intended, all that is left is to generate the systemd service file, enable it and see if it works.

If it does not already exist, create the systemd user directory in your home directory.

```bash
mkdir -p ~/.config/systemd/user
```

Then generate the systemd service file.

```bash
podman generate systemd \
	--user \
	--new \
	--name foo \
	-f /home/user/.config/systemd/user/
``` 

This will create container-jupyter.service in `~/.config/systemd/user/`. If you do not use `--name` you will get a service file named `container-dg6464323fsdf34536`...

After creating the service file, reload systemd.

```bash
Systemctl --user daemon-reload
```

Stop the jupyter container.

```bash
podman stop foo
```

Enable and start container-jupyter.service.

```bash
Systemctl --user enable --now container-foo.service
```

Check if everything worked

```bash
Systemctl --user status container-foo
```

```bash
podman container ps foo
```

If you want the container to be accessible when the user is not logged in, you will need to run 

```bash
loginctl enable-linger user
```

This will allow services owned by the user to run without the user being logged in.
## Conclusion

That is it. Simple and easy. Podman is great for developing and testing Kubernete Pods. Podman can convert Docker or Docker-Compose file to kubernetes yaml.
