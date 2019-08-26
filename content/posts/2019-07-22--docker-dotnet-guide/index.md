---
title: How to Develop a React .NET Core App in a Docker Container Using VSCode
category: "Dev"
cover: photo-docker.png
author: Geoffrey Parry-Grass
---

##Introduction

Developing inside a container has many advantages, including the ability to 
maintain a consistent development environment across a dev team, and easy onboarding.
It also saves the need to manage multiple versions of software on a developer's machine,
and allows easier transitioning of the product to production environments.

Integration of Docker into Visual Studio Code now allows for relatively smooth and seamless
development natively inside the familiar and friendly text editor with minimal overhead.

In the post we will go through the setting up of a basic .Net Core project with react front-end
which will allow us to get started with dockerized development. The principles should also translate
across to other development stacks.

##Prerequisites

The software prerequisites which I will assume you have installed already are Visual Studio Code and 
Docker Desktop. If not they can be easily installed from the links below.

[Visual Studio Code](https://code.visualstudio.com)

[Docker (Windows and Mac)](https://www.docker.com/products/docker-desktop)

[Docker (Linux)](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

##Remote Development VSCode Extension

The native container development integrated into VSCode is provided as part of Microsoft's extension
pack "Remote Development". To enable it simply search for and install this extension pack via the
extension section of VSCode. Once reloading it should now be enabled.

##Creating the Project and Container

Now we are ready to create our containerized project!

First create and open a new folder for your project.

In the bottom left corner of VSCode there should now be the green button for remote development options.

Once clicking on this choose the option to "reopen folder in container"

We are now prompted to choose a starting container to begin development with. If you are not familiar with
containers you can think of this as a very light weight virtual machine with only the software required to
create our app.

For this guide we will chose "C# (.Net Core Latest)". This includes the .Net Core SDK and other requirements.

VSCode will now work with Docker to build our new container and you can monitor its progress in the output window.

Once the container has built we will open up an integrated terminal window within VSCode. This terminal will 
automatically connect to the Docker container so we can run commands within it.

We will now use dotnet to create a new project using a template. The "reactredux" template will create a
ASP.NET Core API and React SPA with Redux state management.

```
dotnet new reactredux
```

Ok, now we are ready to run our project using

```
dotnet run
```

in one terminal window and

```
cd ClientApp
npm start
```

in another

...and we will get an error telling us node is not installed.

##Install Node into the Container

Since containers are designed to be as light weight as possible, the base container does not include Node.
We can tell VSCode to tell Docker to install Node into our container by editing the Dockerfile in the .devcontainer 
directory of our project. To do this we add the following code above the clean up section.

```
&& curl -sL https://deb.nodesource.com/setup_10.x | bash - \
&& apt-get install -y nodejs \
&& apt-get install -y build-essential \
```

The Dockerfile is run when the container is built and is used to configure it. Now Node will be available when the container
is rebuilt. 

The container can be rebuilt by clicking the green remote development icon and choosing "Rebuild Container"

##Configuring Ports and Servers

To allow us to view our application from our browser we will need to configure the ports of our Docker Container.
This allows us to bind our container's ports to our machine, so when we access localhost:3000 it gets routed to the container,
which is running our server.

We can choose which ports are bound by our container in the devcontainer.json file which is next to our Dockerfile.

By uncommenting the "appPort: []," line and adding ports 3000, 5000, and 5001 to the appPort array.

It is also a good idea to set our react server to run separately from our .Net app as described in
[this link](https://docs.microsoft.com/en-us/aspnet/core/client-side/spa/react?view=aspnetcore-2.2&tabs=visual-studio)

The last 2 changes we will need to make is to configure our server routing.
The first is to add a line to our package.json file within the ClientApp directory.
This will cause the calls to our API to be sent from our react server to the dotnet server.
Add the following line before the dependencies line.

```
"proxy": "http://0.0.0.0:5000"
```

The last change is to alter the "applicationUrl" line in the launchSetting.json file (in the Properties directory).
Change the line to

```
"applicationUrl": "https://0.0.0.0:5001;http://0.0.0.0:5000",
```

This is related to the way networking operates in a Docker environment.

##Create a Certificate

Since we are working with SSL Connections we will need to create a certificate (for development only) to allow our app to work.
When the app is deployed a proper SSL certificate will be required.

To create an SSL certificate for development, you can follow a guide such as [this](https://letsencrypt.org/docs/certificates-for-localhost/)

To make the certificate available to the server running in the docker container add this code to the devContainer.json

```
"runArgs": [ "-e", "Kestrel__Certificates__Default__Path=/root/.dotnet/https/localhost.pfx", 
    "-e", "Kestrel__Certificates__Default__Password=[YOUR_CERTIFICATE_PASSWORD]",
    "-v", "[PATH_TO_YOUR_FOLDER_CONTAINING_CERTIFICATE]:/root/.dotnet/https"
],	
```

This will make the certificate available to the Docker container and configure the Kestral development server to use it.
Since it is purely for development hiding the password is not a concern.

##Configure CORS

The final step to setting up our project is to enable CORS.
In a development setting we can allow all hosts for convenience.

If you are not sure how to do this, add the following line to the Program.cs file in the ConfigureServices method:

```
services.AddCors();
```

and 

```
app.UseCors(builder =>
{
    builder.AllowAnyOrigin();
});
```

to the Configure method.

##Conclusion
Now the project is set-up and ready to roll! You can now edit the code in VSCode and run the project from the integrated terminal.

