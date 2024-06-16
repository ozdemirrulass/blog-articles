---
title: "Enterprise-Level Authentication in a Containerized Environment for Next.js 13"
seoTitle: "Enterprise-Level Authentication in a Containerized Environment for Nex"
seoDescription: "Integrate NextJS, Keycloak and MySQL services and handle authentication flow in a containerized environment."
datePublished: Wed Jun 12 2024 15:29:37 GMT+0000 (Coordinated Universal Time)
cuid: clxbzlfii000008jvh2me46ki
slug: enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718205945363/0e4b1360-9498-49c6-9867-7d56319c6f0f.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1718205974412/ee7bac93-9b32-49ae-8bd6-4d77785b53de.png
tags: authentication, docker, mysql, nextjs, oauth2, keycloak, docker-compose, openid-connect

---

## **TL;DR**

[https://github.com/ozdemirrulass/keycloak-nextjs-mysql-docker](https://github.com/ozdemirrulass/keycloak-nextjs-mysql-docker)

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">This tutorial uses Next-Auth and Next-Auth is becoming AuthJS! If you want to use AuthJS for working with latest versions checkout <a target="_blank" rel="noopener noreferrer nofollow" href="https://ulasozdemir.com.tr/enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13-authjs-patch" style="pointer-events: none">THIS POST</a></div>
</div>

This article aims to provide a step by step guide for Keycloak + NextJS 13 authentication and containerization. By the end of this article you will be able to

1. Set up a Keycloak server based on MYSQL database for authentication.
    
2. Integrate Keycloak with a Next.js 13 application for user authentication.
    
3. Implement authentication flows such as login, registration in your Next.js app.
    
4. Containerize your Keycloak and Next.js applications using Docker for easy deployment and scalability.
    
5. Understand best practices for managing authentication tokens and sessions in a containerized environment.
    

From start to end we will be using compose to build our application in a containerized environment. Since we will be building our containers for development environment, output source code of this tutorial will be ready to convert multi-environment. Since the requirements and configurations of production and development environments are different, Multi-environment development is strongly advised but we will be only working on development environment for educational purposes.

## Prerequisites

[I](https://github.com/vercel/next.js/tree/canary/examples/with-docker-multi-env) did my best to keep this article as simple as possible to teach the basics but to get the most out of this article it is strongly advised to have some knowledge on Next.js, Docker, and authentication concepts.

Tools and Software: Docker, TypeScript, NodeJS, Docker Compose, Keycloak

## Setting Up Development Environment

Let's start creating a new project directory named `keycloak-nextjs-docker-tutorial` and open it using your favorite IDE. I'll be using VS Code.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717976952288/3f9a55c2-bf87-47a4-b134-eed5af035ff4.png align="center")

As I mentioned earlier we will be using docker compose tool to create our containers.

For those unfamiliar with docker compose tool:

It is simply **a tool for defining and running multi-container applications**. To be able to run containers we must define the properties of our containers in a `yml` file.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">By default, Docker Compose looks for a file named <code>docker-compose.yml</code> or <code>docker-compose.yaml</code> in the current directory. However, you can specify a different file using the <code>-f</code> or <code>--file</code> option when running Docker Compose commands.</div>
</div>

For example, if your `docker-compose.yml` file is named `my-compose.yml`, you can use the following command to specify the file:

```bash
docker-compose -f my-compose.yml up
```

We will be building a development environment so let's create a file named `docker-compose.dev.yml` to define our container properties and `.env` file to specify some environment values.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717978835215/aed3a3fd-ecaf-4a98-b6a0-a43ebcbd46d8.png align="center")

Typical `docker-compose.yml` file contains the following definitions:

* **Version**: This field specifies the version of the Docker Compose file format being used. It's usually first line of the file and defines the schema of the file.
    

```yaml
version: '3.8'
```

* **Services**: This section defines the containers that make up your application. Each service must be identified by a unique name, and its configuration must be under that name.
    

```yaml
services:
    keycloak:
        ...
    mysql:
        ...
    next-app:
        ...
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Be careful while working on YAML or YML files. YAML and YML files uses indentation to indicate the structure of the data. The recommended indentation is <strong>two spaces per level </strong>but as long as it is consistent across the file it can be 2-3 or anything.</div>
</div>

* **Container Configuration**: Within each service definition, you can configure various aspects of the container, such as:
    
    * **Image**: Specifies the Docker image to use for the container.
        
    * **Build**: Specifies the path to the Dockerfile if the image needs to be built.
        
    * **Ports**: Maps ports from the container to the host machine.
        
    * **Volumes**: Mounts volumes from the host machine into the container.
        
    * **Environment Variables**: Sets environment variables for the container.
        
    * **Dependencies**: Defines dependencies on other services within the same `docker-compose.yml` file.
        
    * **Networks**: Configures networking options for the container.
        
    * **Command**: Overrides the default command specified in the Docker image.
        
    * **Healthcheck**: Configures a health check for the container.
        

Having said that let's start to configure our containers.

## MySQL

Let's start with the database. We need a database for our Keycloak service to store user data. I'll be using MYSQL for this tutorial but after completing this tutorial I encourage you to try to build the same structure with different database technologies.

```yaml
version: '3.8'

services:
  mysql:
    container_name: mysql
    image: "mysql:${MYSQL_VERSION}"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    environment:
      MYSQL_DATABASE: keycloak
      MYSQL_USER: keycloak
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - ./mysql_data:/var/lib/mysql
```

We've defined a MYSQL container named `mysql` and a database named `keycloak`. User credentials of keycloak database are username: keycloak, password: password and the root user's password is also password.

<div data-node-type="callout">
<div data-node-type="callout-emoji">🔗</div>
<div data-node-type="callout-text">It is strongly advised to check <a target="_blank" rel="noopener noreferrer nofollow" href="https://hub.docker.com/_/mysql" style="pointer-events: none">https://hub.docker.com/_/mysql</a> and learn more about <strong>MySQL environment variables.</strong></div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">I'd like to draw your attention to the usage of <code>${MYSQL_VERSION}</code> in the Docker Compose file. This is an example of variable substitution or interpolation.<code>${VARIABLE_NAME}</code> syntax is used to reference environment variables defined either in the shell environment or in an <code>.env</code> file. Docker Compose automatically detects the <code>.env</code> file within the same directory.</div>
</div>

Open the `.env` file we created earlier in the `/keycloak-nextjs-docker-tutorial` directory and add the following variable.

```plaintext
MYSQL_VERSION=8.0
```

I'll be using [mysql:8.0](https://hub.docker.com/layers/library/mysql/8.0)

Another very important part of this service definition is:

```yaml
volumes:
      - ./mysql_data:/var/lib/mysql
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">This setup is commonly used to persist data generated by the MySQL database even if the container is stopped or removed. It ensures that data stored within the container's <code>/var/lib/mysql</code> directory is stored on your local machine and remains accessible across container restarts or recreations.</div>
</div>

## Keycloak

In this part we will be adding the Keycloak service to our compose definitions and ensure the connectivity between our database.

Open the `docker-compose.dev.yml` file we created earlier in the `/keycloak-nextjs-docker-tutorial` and add the following definitions after the `mysql` service.

```yaml
keycloak:
    container_name: keycloak
    image: "quay.io/keycloak/keycloak:${KC_VERSION}"
    command: ["start-dev"]
    restart: unless-stopped
    depends_on:
      - mysql
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080"]
    environment:
      - KC_DB=mysql
      - KC_DB_USERNAME=keycloak
      - KC_DB_PASSWORD=password
      - KC_DB_URL=jdbc:mysql://mysql:3306/keycloak
      - KC_FEATURES=${KC_FEATURES}
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=${KC_PASSWORD}
    ports:
      - ${KC_PORT}:8080
```

As you can see, we have a few variables here, just like in our mysql service. Let's quickly open the `.env` file we created earlier in the `/keycloak-nextjs-docker-tutorial` directory and add the related variables.

```plaintext
KC_VERSION=17.0.1
KC_PORT=8080
KC_FEATURES=account2,admin2,account-api,token-exchange
KC_PASSWORD=keycloak
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">🔗</div>
<div data-node-type="callout-text">Visit <a target="_blank" rel="noopener noreferrer nofollow" href="https://quay.io/repository/keycloak/keycloak" style="pointer-events: none">https://quay.io/repository/keycloak/keycloak</a> to check other versions of Keycloak.</div>
</div>

There is another very important feature of Docker I'd like to draw your attention.  
`- KC_DB_URL=jdbc:mysql://mysql:3306/keycloak`

Here we specify a database connection url using Java Database Connectivity API but as you can see instead of using `localhost` or a domain or an ip address we are using the container name!

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">When using Docker Compose to define multiple services, Docker Compose creates a <strong>bridge network</strong> by default and assigns a unique name to each container <strong>within that network</strong>. By specifying a <strong>container name</strong> in Docker Compose, you are giving that container a human-readable alias within the context of the <strong>Docker network</strong>.</div>
</div>

Good job! 👏 We've just prepared a Docker Compose environment for Keycloak based on MYSQL database.

Let's pause here, run our containers, and take a closer look at what we have accomplished.

To build and run our containers execute the following command in the same directory with `docker-compose.dev.yml` file which in our case `/keycloak-nextjs-docker-tutorial`.

```bash
docker compose -f docker-compose.dev.yml up -d
```

Before executing the code I'd like to explain the `-f` and `-d` flags.

* `-f` flag stands for "file." as I mentioned earlier if your compose file is not docker-compose.yml/yaml you must specify the filename.
    
* `-d` flag stands for "detached mode". When you use this flag, Docker Compose will run the containers in the background, allowing you to continue using the terminal for other tasks. If you don't use **\-d** flag containers will start in the foreground, and the logs from the containers will be streamed to your terminal. This means you'll see the output of each container's STDOUT and STDERR directly in your terminal window.
    

If you followed the previous steps precisely you must be seeing something like this in your terminal:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717996803152/8b8baf4f-ba40-4613-835b-c40090bbbac0.png align="center")

P.S. Container names must be unique and since I already have a container named mysql on my system I named my MYSQL container as mysql.

To see which containers are running open your terminal and execute

```bash
docker ps
```

You'll see two containers:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717998024074/5806eab9-53ec-4bfd-b79e-08d1185bae39.png align="center")

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><code>0.0.0.0:8080-&gt;8080/tcp</code> This is another important feature of docker and we mentioned it earlier but It's worth repeating. It represents the port mapping. This indicates that port 8080 on the host machine is mapped to port 8080 on the container. This means that any traffic directed to port 8080 on the host machine will be forwarded to the corresponding port on the Docker container. This means when you try to reach to service from your host machine you must use the port shows up in the left side but if another service in the same network tries to reach this service it should use the port in the right side of <code>:</code>. It is simply <code>[hostPort]:[containerPort]</code></div>
</div>

Let's check if our Keycloak and MySQL integration is working.  
Visit [localhost:8080](http://127.0.0.1:8080) on your web browser. You should see the following screen:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717998960501/f7f54985-e77d-4fbb-8fbb-d1c61634fe38.png align="center")

Seems like Keycloak running fine. Let's login with the credentials we've defined in `.env` file earlier. Click to Administrator console and login with the following credentials:

```plaintext
Username: admin
Password: keycloak
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717999091441/da5ae807-666a-4b5a-9dd2-fa20be120577.png align="center")

You will see a Keycloak dashboard after successful login

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717999247729/f1747468-0c9a-40b7-9199-14781bc7bb1b.png align="center")

Perfect! Before we proceed further, let me demonstrate how to access container terminals and execute commands within Docker environments.

```bash
docker exec -it <container_id_or_name> <command>
```

This is the syntax of executing a command in a container. For example, if you have a container running a bash shell and its ID is `abcdef123456`, you can access its terminal like this:

```bash
docker exec -it abcdef123456 bash
```

This command opens an interactive terminal (`-it`) within the specified container (`abcdef123456`) running the `bash` shell. You can replace `bash` with any other command you want to execute within the container.

We will be working inside mysql container so it must be:

```bash
docker exec -it mysql bash
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">If you named your containers different you can check the container names executing <code>docker ps</code> command in your terminal.</div>
</div>

After executing the bash command user@host indicator will replace with `bash-5.1#` like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717999973079/b5ced7ef-6e1a-445c-8b99-c5eb5703b036.png align="center")

This means that you've successfully accessed the bash shell of your container. Now Let's connect to a MySQL database and check tables and records.

```bash
mysql -u keycloak -p
```

after you execute this command in the bash shell it will require you to enter a password which we specified in `docker-compose.dev.yml` as

```yaml
MYSQL_USER: keycloak
MYSQL_PASSWORD: password
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718000282890/0eae668a-a208-4b60-b4e4-ba6346396b69.png align="center")

Let's check the databases exists in our mysql service.  
Execute the following query in MySQL monitor

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718000412566/2a709785-69de-412b-812d-7f11596168b2.png align="center")

You will see 3 database record. 2 of them are default special databases of MySQL. The `information_schema` database provides metadata about MySQL server objects, while the `performance_schema` database offers insights into server performance metrics.

Important part for us is keycloak database. It is perfectly created just like we defined as `MYSQL_DATABASE: keycloak` in our compose file.  
To be able to see the the contents execute the following commands in order.  
`use keycloak;`  
`show tables;`

This will show us a list of tables which our Keycloak service created automatically.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718000878504/07c3934c-fc8c-4bc7-a6d5-47f0fa5f27e9.png align="center")

Execute `exit;` to exit from the MySQL monitor and execute `exit` in the bash shell to exit from the MySQL container's bash shell.

👏 Congratulations on our achievements so far! Yet, there's still more to accomplish!

Before we proceed, I'd like to introduce the concept of Multi-tenancy. Understanding the Multi-tenancy concept will help you to understand and operate Keycloak better!

A "tenant" typically refers to an individual or organization that has its own distinct set of users, data, and configuration settings within the shared software environment.

Multitenancy refers to a software architecture where a single instance of the software serves multiple clients (tenants), keeping their data and configurations separate while sharing the same underlying infrastructure.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718002436749/dd3236ca-1865-4ba8-a8f2-1e4b63496586.jpeg align="center")

I strongly advise you to read more on multi-tenancy and understand the concept and types of it.

<div data-node-type="callout">
<div data-node-type="callout-emoji">🔥</div>
<div data-node-type="callout-text">In Keycloak's architecture, <strong>Realms</strong> essentially serve as the tenants.</div>
</div>

There is a beautiful explanation of [**handling multitenant organization with Keycloak**](https://documentation.cloud-iam.com/how-to-guides/multitenant-with-keycloak.html) in the documentation of cloud-iam.

We will go for second for simplicity and consistency reasons.

Let's create our first REALM.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003253694/ebdb2a3e-39ed-478a-885a-220fd4d17fdd.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003310191/07cab136-927d-475e-8b42-5ac390d7fe06.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003456206/0ecedb5f-e13d-4c74-a3b7-960614490694.png align="center")

Perfect! We just created our first REALM. Let's create a client for our NextJS application now!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003607091/aa68fcf1-d802-487e-b238-1f76cffc2ce9.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003695095/0985e94f-b376-4306-b6c8-448abfe3c620.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718003969980/55483b15-62b6-46be-aee9-aefdb27ae0f7.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718004195388/66ef1029-ea43-4bf1-bd99-841f05848114.png align="center")

If all goes well you must see the following screen:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718004269083/0b9581b5-f978-43b1-ae10-256dd42a521c.png align="center")

Congrats 🥳 We have a Realm and a Client for our NextJS application.

Before we proceed further with Client configuration and Access Settings we will create and containerize a NextJS application! 💃🕺

> “If you're not a disruptor, you will be disrupted.” – John Chambers

So;

> "Mr. Gorbachev, tear down this wall!"- Reagan

Open your terminal in the root of our project which is `/keycloak-nextjs-docker-tutorial` and execute this command:

```bash
docker compose -f docker-compose.dev.yml down
```

This command will stop and remove all of the containers defined in our compose file as well as the docker network.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718006105699/79a623f4-925a-4fe2-84b2-01c83e358d2a.png align="center")

Did we made all this for nothing !? Of course NO!  
Open the project directory in your favorite IDE and take a look at the folder structure of the project.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718006309981/c7494689-38bf-45fa-b257-2ab5ca98fb3f.png align="center")

Docker Compose and our MySQL service left us a little present. Well, actually we wanted this from them...  
Do you remember this part of the `docker-compose.dev.yml` file?

```yaml
    volumes:
      - ./mysql_data:/var/lib/mysql
```

Thanks to this piece of definition whenever we built our service again mysql will mount the latest status and contents of our databases.

We will reunite with out precious data but first Let's create our NextJS application and containerize it!

## NextJS

Open the project directory in your terminal `/keycloak-nextjs-docker-tutorial` and execute the following command to create a NextJS app:

```bash
npx create-next-app@latest
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718007977133/0e41fdab-209c-4eb4-8ba6-d5235fab4672.png align="center")

Once again we will open the project folder using our IDE.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718008085570/ee43fd78-9dec-4799-944f-fab10d224ffc.png align="center")

Here is our next application. Isn't it adorable? No! Because it is not containerized. Let's get to work then!

But before we work on our application we should add the `/next-app` directory to workplace otherwise ESLint will throw errors.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718009284241/d61bf96d-ed58-4f8f-99c7-044882e55b02.png align="center")

File-&gt;Add Folder to Workspace-&gt;(Choose next-app directory) and click add.  
Now lets open `next.config.mjs` file in our `/next-app` directory

Next we add a property inside of the nextConfig const.

```javascript
output: "standalone",
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718009736383/cb6b165d-b437-4ab1-afe5-6b0ebdfc1247.png align="center")

This declaration is to create a self-contained build of our application. This can be particularly useful for deploying our Next.js application in environments where we want to avoid installing Node.js dependencies directly on the server, such as when using Docker or deploying to a serverless environment. We will see the benefit of this when we want to write a production environment.

Until now we built and run an existing images by pulling from their resources. This time we will create a docker image for our NextJs Application.

First create a new file as `dev.Dockerfile` inside the `/next-app` directory and open the file on IDE.

Before writing anything let's first understand what is Dockerfile then break it down it together.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image. It has a simple syntax and consists of a series of instructions that are executed in order to build a Docker image.</div>
</div>

```dockerfile
# Base Image: Typically, a Dockerfile starts with a base image upon which
# you build your application. 
# This is specified using the FROM instruction.
FROM node:18-alpine

# If you set WORKDIR /app, then any commands or file operations 
# will be relative to the /app directory.
WORKDIR /app

# Install dependencies based on the preferred package manager
# Copy Application Files: You copy the application code or files
# into the image using the COPY or ADD instruction.
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i; \
  # Allow install without lockfile, so example works 
  #even without Node.js installed locally
  else echo "Warning: Lockfile not found. It is recommended to commit lockfiles to version control." && yarn install; \
  fi

COPY . .

# Next.js collects completely anonymous telemetry data about 
# general usage. Learn more here: https://nextjs.org/telemetry
# Comment the following line to enable telemetry at run time
ENV NEXT_TELEMETRY_DISABLED 1

# Note: We could expose ports here but instead
# Compose will handle that for us

# Start Next.js in development mode based on the 
# preferred package manager
CMD \
  if [ -f yarn.lock ]; then yarn dev; \
  elif [ -f package-lock.json ]; then npm run dev; \
  elif [ -f pnpm-lock.yaml ]; then pnpm dev; \
  else npm run dev; \
  fi
```

Beautiful.

Now we will add this Dockerfile to our Compose definitions so that Compose tool will be able to build the image for us just as we defined in our `Dockerfile`. Add the following block under services in `docker-compose.dev.yml`.

```yaml
next-app:
    container_name: next-app
    build:
      context: ./next-app
      dockerfile: dev.Dockerfile
    restart: unless-stopped
    environment:
      - NODE_ENV=development
    volumes:
      - ./next-app:/app
      - /app/node_modules
    ports:
      - 3000:3000
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">context definition under the build parameter defines the location of our Dockerfile.</div>
</div>

Before building and running the containers I'd like to mention one more concept.

**Network Isolation:** until now we've been using the default bridge network which Docker built for us. But let's think on this for a second.  
While working on a multi-service environment docker automatically creates a bridge network and assigns each container **within that network with a unique name**.

What does this mean?

Does it mean when we add NextJS application in this compose file will mysql can be accessible by NextJS ?

Answer is Yes! But do we want it though? What are we supposed to do with MySQL in a NextJS front-end client? Nothing. Let's cut the bonds then!

What we have to do is isolating the connectivity between our services like in this diagram by creating custom networks:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718016228889/5db69400-6a3e-4cfa-8d6f-00b0d07dc4e3.png align="center")

open `docker-compose.dev.yml` once again and go to very bottom of it and add following piece of definition.

```yaml
networks:
  frontend-network:
  keycloak-network:
```

By adding this definition, we've created 2 new networks for our environment. As you can see I did not specify any property for our networks because the default network type is `bridge`.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Bridge type networks allows containers to communicate with each other, and also provides external access if ports are exposed.</div>
</div>

and this is exactly what we want.

Adding the previous piece of definition in our compose file only creates the network. To be able to use it we must introduce our containers to our new networks.

```yaml
version: '3.8'

services:
  mysql:
    container_name: mysqlk
    image: "mysql:${MYSQL_VERSION}"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    environment:
      MYSQL_DATABASE: keycloak
      MYSQL_USER: keycloak
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - ./mysql_data:/var/lib/mysql
    networks:
      - keycloak-network
    
  keycloak:
    container_name: keycloak
    image: "quay.io/keycloak/keycloak:${KC_VERSION}"
    command: ["start-dev"]
    restart: unless-stopped
    depends_on:
      - mysql
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080"]
    environment:
      - KC_DB=mysql
      - KC_DB_USERNAME=keycloak
      - KC_DB_PASSWORD=password
      - KC_DB_URL=jdbc:mysql://mysqlk:3306/keycloak
      - KC_FEATURES=${KC_FEATURES}
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=${KC_PASSWORD}
    ports:
      - ${KC_PORT}:8080
    networks:
      - keycloak-network
      - frontend-network

  next-app:
    container_name: next-app
    build:
      context: ./next-app
      dockerfile: dev.Dockerfile
    restart: unless-stopped
    environment:
      - NODE_ENV=development
    volumes:
      - ./next-app:/app
      - /app/node_modules
    ports:
      - 3000:3000
    networks:
      - frontend-network
  
networks:
  frontend-network:
  keycloak-network:
```

As you can see in the current version of our compose file, I added the networks to the bottom of each service.

It's time to see what we've done. Open the terminal in the root directory of our project and run the following command to build image.

```bash
docker compose -f docker-compose.dev.yml build
```

This will build the images you defined in the Compose file.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">You must re-build the image for any changes in Dockerfile.</div>
</div>

It's time to run our containers.

```bash
docker compose -f docker-compose.dev.yml up -d
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">If you run the <code>up</code> command without using the <code>build</code> command, Compose will check if a pre-built image with the same name exists. If it does, Compose will use that image. If it doesn't, Compose will build the image first and then start the container with it.</div>
</div>

It will take a bit more time for Keycloak to start comparing other services. In the meanwhile we can check if our next-app.  
Visit `localhost:3000` on your browser.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718187866002/be5cda9a-cbdc-49dd-84e6-a8d67563033a.png align="center")

Nice work!  
Let's make sure our hot load works as expected and our container applies the changes we make in our host machine through the volume.

Open `/next-app/src/app/page.tsx` and replace the content with the following code:

```typescript
export default function Home() {
  return (
    <main>
        <div>It Works!</div>
    </main>
  );
}
```

Visit `localhost:3000` and you must see the changes!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718197529502/95f4d3e6-9390-4049-911f-a051e1c8371f.png align="center")

Now it's time to integrate Keycloak authentication for our our NextJS application.  
We will be using [NextAuth](https://next-auth.js.org/).

Install Next-Auth package:

```bash
npm install next-auth
```

Create a new directory under `/next-app` directory as `types` and a file inside types as `node-env.d.ts` .

```typescript
// /next-app/types/node-env.d.ts
declare namespace NodeJS {
    export interface ProcessEnv {
      NEXT_PUBLIC_KEYCLOAK_CLIENT_ID: string
      KEYCLOAK_CLIENT_SECRET: string
      NEXT_LOCAL_KEYCLOAK_URL: string
      NEXT_PUBLIC_KEYCLOAK_REALM: string
      NEXT_CONTAINER_KEYCLOAK_ENDPOINT: string
    }
  }
```

now it's time to create authentication routes.

```typescript
import { AuthOptions } from "next-auth";
import NextAuth from "next-auth/next";
import KeycloakProvider from "next-auth/providers/keycloak";

export const authOptions: AuthOptions = {
  providers: [
    KeycloakProvider({
      jwks_endpoint: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/certs`,
      wellKnown: undefined,
      clientId: process.env.NEXT_PUBLIC_KEYCLOAK_CLIENT_ID,
      clientSecret: process.env.KEYCLOAK_CLIENT_SECRET, 
      issuer: `${process.env.NEXT_LOCAL_KEYCLOAK_URL}/realms/${process.env.NEXT_PUBLIC_KEYCLOAK_REALM}`,
      authorization: {
        params: {
          scope: "openid email profile",
        },
        url: `${process.env.NEXT_LOCAL_KEYCLOAK_URL}/realms/myrealm/protocol/openid-connect/auth`,
      },      
      token: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/token`,
      userinfo: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/userinfo`,
    }),
  ],
  
};
const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

Go back to root directory of your next-app and create an environment file as `.env.local`

```typescript
NEXT_PUBLIC_KEYCLOAK_REALM=<realm-name>
NEXT_PUBLIC_KEYCLOAK_CLIENT_ID=<client-name>
KEYCLOAK_CLIENT_SECRET=<secret-from-keycloak-client>
NEXTAUTH_SECRET=<create-using-openssl>
NEXT_LOCAL_KEYCLOAK_URL="http://localhost:8080"
NEXT_CONTAINER_KEYCLOAK_ENDPOINT="http://keycloak:8080"
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">For local network connectivity purposes we will use keycloak url and jwks endpoint differently from each other but in most cases in a development environment both should be same.</div>
</div>

As I mentioned before Keycloak still has the Realm and the Client we've created before.

Visit `localhost:8080` and sign in and select the Realm we created earlier:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718199360592/9ac4bebd-a0d5-42db-a470-afef46b1e8d3.png align="center")

Go to clients and select the client we created earlier.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718199456303/b37f8d36-56b6-40b4-9b18-b0bf9bd9bfaf.png align="center")

Go to Access Settings and define Home URL and Valid redirect URI's:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718199612083/e0959af4-c65c-4783-82d4-f9a8a002b371.png align="center")

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Valid URI pattern a browser can redirect to after a successful login or logout. Simple wildcards are allowed such as '<a target="_blank" rel="noopener noreferrer nofollow" href="http://example.com/*" style="pointer-events: none">http://example.com/*</a>'. Relative path can be specified too such as /my/relative/path/*. Relative paths are relative to the client root URL, or if none is specified the auth server root URL is used. For SAML, you must set valid URI patterns if you are relying on the consumer service URL embedded with the login request.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Home URL: Default URL to use when the auth server needs to redirect or link back to the client.</div>
</div>

Save and go to Credentials.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718199750275/5778b2c8-350c-4f19-b124-e91df84f7eef.png align="center")

Copy Client secret and paste `KEYCLOAK_CLIENT_SECRET` value.

```plaintext
KEYCLOAK_CLIENT_SECRET=<secret-from-keycloak-client>
```

For `NEXTAUTH_SECRET`***,*** create a secret by running the following command and add it to `.env.local`. This secret is used to sign and encrypt cookies.

```typescript
openssl rand -base64 32
```

In my case final `.env.local` file looks like this:

```plaintext
NEXT_LOCAL_KEYCLOAK_URL="http://localhost:8080"
NEXT_CONTAINER_KEYCLOAK_ENDPOINT="http://keycloak:8080"
NEXT_PUBLIC_KEYCLOAK_REALM="myrealm"
NEXT_PUBLIC_KEYCLOAK_CLIENT_ID="next-app"
KEYCLOAK_CLIENT_SECRET="71ikzeN5p0fEwdHW6Hw5jOmlRvRIEtgO"
NEXTAUTH_SECRET="MdiNiCNlDcBP8fUmANd9ARPIB+tlKV/oy3m88W2bTHk="
```

Create a new folder under `/next-app/src` as `components` and create the following components inside the directory.

```typescript
//next-app/src/components/Login.tsx

"use client"
import { signIn } from "next-auth/react";
export default function Login() {
  return <button onClick={() => signIn("keycloak")}>
    Signin with keycloak
  </button>
}
```

```typescript
//next-app/src/components/Logout.tsx

"use client"
import { signOut } from "next-auth/react";
export default function Logout() {
  return <button onClick={() => signOut()}>
    Signout of keycloak
  </button>
}
```

Go to page.tsx file `/next-app/src/app/page.tsx` and replace the content with the following block:

```typescript
import { getServerSession } from 'next-auth'
import { authOptions } from './api/auth/[...nextauth]/route'
import Login from '../components/Login'
import Logout from '../components/Logout'
export default async function Home() {
  const session = await getServerSession(authOptions)
  if (session) {
    return <div>
      <div>Your name is {session.user?.name}</div>
      <div><Logout /> </div>
    </div>
  }
  return (
    <div>
      <Login />
    </div>
  )
}
```

We need a user to test our client.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718201864382/a4f33006-c6ca-40ab-b31c-731ebbdabc60.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718201782220/9f5cb3da-2ddc-43ae-a726-8342b32d94e3.png align="center")

Create the user and go to `Credentials` section to set a password for the user we just created.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718202109031/9a9817c1-8975-4eb7-992e-2057ef0dd040.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718202158508/b725c626-09f4-4003-bb9d-1bad075cdc26.png align="center")

Now its time to test our authentication flow.

Go to our nextjs app `localhost:3000`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718202269986/d5ad989b-d551-4c87-b028-0bb8fbecf05c.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718202309240/d35f975f-f68e-4ba3-83f2-0fd5d5c8ab6e.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718203784258/d2b33b43-2260-4de2-b42f-57f2bf22f0c6.png align="center")

Congratulations we have successfully implemented an enterprise-level authentication in a containerized environment for Next.js 13 front-end.

For federated logout you my want to check this discussion on github: [https://github.com/nextauthjs/next-auth/discussions/3938](https://github.com/nextauthjs/next-auth/discussions/3938)

And here is how you can activate user registration:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718204312528/38095373-d541-4a3e-824e-a2f01b5607a4.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718204414500/56a3b50a-c31e-4fda-bc21-6de488950a3c.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718204439212/2255cd7b-8d1b-4782-bd88-bac56fa9f90c.png align="center")

I'll also write how to integrate it with your backend service and how to monitor your application and keycloak using Prometheus and Grafana as soon as possible. It will also cover securing your Next.js routes using Keycloak's role-based access control.

### Thank you, see you soon!