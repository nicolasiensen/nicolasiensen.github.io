---
layout: post
title:  "Creating a new Rails application with Docker"
share-description: This tutorial explains how to bootstrap a new Rails application without having Ruby installed and relying entirely on Docker.
tags: ruby rails docker tutorial
---

There are many tutorials you can Google on how to *dockerize* an existing Rails application, but what about creating a new Rails application from scratch using Docker?

That is the question I wanted to answer while bootstrapping a new pet project with Rails. My goal was to successfully run the command `rails new` without having Ruby installed in my host machine and, of course, run the application on a container.

The only thing you will need to be installed in your machine to follow this tutorial is Docker. Please find out how to install it in your platform at [docs.docker.com/get-docker](https://docs.docker.com/get-docker/).

## Step 1: Create the folder for the new Rails application

First, open a terminal and type the commands below that will create and navigate inside the new folder for your project:

```sh
mkdir my-app
cd my-app
```

If you are using Windows, you might have to use different commands to achieve the same result.

## Step 2: Create the Docker files

Once inside the new folder, we create the file `Dockerfile` with a minimal setup to create an image that supports Ruby:

```dockerfile
FROM ruby:3.1.0

WORKDIR /usr/src/app
```

Before building the new Docker image, let's create a file named `docker-compose.yml` with the following content:

```yml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/usr/src/app
```

The file above sets up a container called `web` that exposes the port 3000 (the port used by Rails) and sets up a volume that mounts the current path of the host machine to the folder `/usr/src/app` in the container.

The volume is essential so that when we generate the Rails application in the container, the template files will persist in the host filesystem.

## Step 3: Generate a Rails application

Now we can go ahead and access the terminal of the container while exposing the service ports so we can access the Rails application later on via localhost:3000:

```sh
docker-compose run --service-ports web bash
```

Once inside the container, we can install Rails and generate a new application:

```
gem install rails
rails new .
```

After running these commands, you should see all the files generated by Rails in the project folder.

## Step 4: Start the application

Next, we can start our Rails application using the option `-b` to bind the webserver to the correct IP address.

```
rails s -b 0.0.0.0
```

If everything goes well, we should be able to access [localhost:3000](http://localhost:3000/) and see the default homepage of our new Rails application:

![Rails application default homepage](/assets/img/posts/2022-02-01-creating-a-new-rails-application-with-docker/rails-app-homepage.png)

## Step 5: Install project dependencies when building the Docker image

We generated the new Rails application, and we can start it from inside the container, but if you exit the container and reaccess it, you will see that we can no longer start the server.

1. Press `ctrl+c` to stop the server
1. Type `exit` to exit the container
1. Type `docker-compose run --service-ports web bash` to start a new terminal session
1. Type `rails s -b 0.0.0.0` to start the Rails application

The terminal will show an error message that it can't find the command `rails`. This error happens because we installed Rails in the previous terminal session, and it doesn't persist between sessions.

To fix that, we can change the file `Dockerfile` to install all the project dependencies, including Rails as part of the build process:

```dockerfile
FROM ruby:3.1.0

WORKDIR /usr/src/app

COPY . .
RUN bundle install
```

The last two lines copy all the files from the current folder of the host machine to the Docker image, including the Gemfile, and run the command to install all the project dependencies.

From the host machine, we can now run `docker-compose build` to build the new image that will contain all dependencies installed.

After that, we can then access a new terminal session in the Docker container and start the Rails application:

```sh
docker-compose run --service-ports web bash
rails s -b 0.0.0.0
```

To confirm that everything is working, you can access [localhost:3000](http://localhost:3000/) and verify that the default Rails homepage shows up.

## Step 6: Set up a default command for the main container

Before we go, let's define a command for our main container, so we can start the Rails application by running `docker-compose up`:

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/usr/src/app
    command: rails s -b 0.0.0.0
```

If you now run `docker-compose up` from the host machine, the Rails application will start, and you should be able to access it at [localhost:3000](http://localhost:3000/).

With that, you are ready to commit and share the bootstrap of your Rails application with your coworkers.

## Conclusion

I have two computers at home I use for coding, and none of them has any programming language installed. Instead, I rely on Docker to encapsulate and standardize the development environment for all my projects.

This approach allows me to get rid of tools such as [rvm](https://rvm.io/) to manage different Ruby versions on my machine, and it ensures that I share the same development environment as my coworkers.