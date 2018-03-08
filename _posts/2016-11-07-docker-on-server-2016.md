---
layout: post
comments: true
title: Docker on Server 2016
tags:
  - Windows
  - Docker
categories:
  - Docker
---

Now that Server 2016 has launched Windows Containers are a thing. I've been busy shoehorning as many things as possible into containers,
what follows are observations and tips based on my experience so far.

* Study the Dockerfile reference, in particular the difference between shell and exec commands, and the use of CMD and ENTRYPOINT.
* Chocolatey is your friend, much easier than running random MSI packages. You can use Chocolatey in your Dockerfile like this:

```dockerfile
FROM microsoft/windowsservercore:latest

ENV chocolateyUseWindowsCompression false

RUN powershell iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

RUN ["choco", "install", "javaruntime", "-y"]
```

* When your container runs, whatever process is started by CMD or ENTRYPOINT needs to stay active, as that is what Docker monitors, and the container will stop when that process ends. If your app runs as a service you can use the Wait-Service script to monitor it.
* Microsoft recently released documentation on how to run a container as an Active Directory account. This is huge as it removes the need to handle credentials in your container, and is probably the one big advantage of Windows containers over Linux.
* By default containers run in a NAT-ed network from the host, but you can create a transparent network so they are exposed directly. Beware if you are creating lots of containers they are created with different MAC addresses each time so you can use up your pool of DHCP addresses. You may want to give them static IPs or static MAC addresses.
* If you want to run multiple containers and access them over the same port (e.g. multiple web apps over port 80) you will need a reverse proxy to route traffic to the correct containers. You can run this on the host itself or in a container. I have created an example using IIS.
* If you want to manage docker remotely you can either use PowerShell remoting to the host and run Docker commands, or you can access the docker daemon directly over a TCP port by adding the following switch to the service command line -H tcp://0.0.0.0:2375 - be aware this makes the host accessible to anyone on the network, you should protect the socket.
* Although docker.exe is a traditional Windows exe it is very PowerShell friendly. The commands either return easily manipulated strings or JSON. Some useful examples

# Container information

```powershell
docker container inspect mycontainer | ConvertFrom-Json
```

# Clean up dangling images

```powershell
docker images -f "dangling=true" -q | %{docker rmi $_}
```

# Stop and remove all containers by label

```powershell
docker container ls -a -f label=mylabel --format {% raw  %}"{{.ID}}"{% endraw %} | % {
    docker container stop $_ } | % {
    docker container rm $_ }
```

That's all that springs to mind right now, but Windows containers are already showing a lot of promise, and there will be plenty more improvements to come.