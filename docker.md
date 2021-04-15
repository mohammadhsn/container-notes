# Containers



## Initializing containers

Creating a container and starting it up are 2 separate processes

`docker create` creates a container (without running it)

`docker start` actually runs a created container

Fortunately, `docker run` command runs both commands immediately

`docker run` => `docker create` + `docker start`

the `docker start` command is not gonna show the container outputs, however the `docker run` command does that. If we gonna see the output in `docker start` command, there is `-a` flag to do this:

```shell
docker start -a
```

`docker start` command just starts (or restarts) an existing (but stopped) container. Note that the command just start it and there is no way to replace the starting command. Lets see an example:

`docker run busybox echo hi there`

Every container has a default command. It means when the containers start, default command immedietly runs inside the container. the abow command creates busybox container from it's image and replaces it's default start command with `echo hi there`. because the echo commands runs in a moment and the container default process gets down, docker stops the container.

If we want to restart the container again, `docker start` command is the solution, **but** the `start` command cannot replace the default command again, so it just restarts the container. Obviously the `echo hi there` executes again.

If you missed the output of container, or you wanna to see more details, don't worry, just try `docker logs` command.



## Terminating containers

There're two major commands to terminate the containers, `docker stop` and `docker kill`.

The `docker stop` command simply sends a unix `SIGTERM` signal to get down the process (container) gracefully.

The `docker kill` command sends a `SIGKILL` signal to force the process to get down immediately.

*Note that `docker stop` command waits for the process to getting down in 10 seconds, if the process doesnt get down, docker kills it immediately.*



## Interacting with containers

Imagine we have a running container and we want to do something with it. For example we have a running **redis** container and we want to explore on it. first lets run it with `docker container run redis`. Now we gonna to connect to redis through redis client and set a new key in redis, but the client is available inside the container, so we need to run something **inside** our running container.

```shell
docker exec -it my_redis_container_id redis-cli
```

If you need to a unix shell in order to prevent repeat the `docker exec` command, you may want to use `sh`:

```shell
docker exec -it my_redis_conatainer_id sh
```



# Images



## Building our custom images

We already used creating containers from existing images from docker hub, now we gonna create our custom imagas, to do this let's create a file called *Dockerfile* in somewhere of filesystem and write some lines of commands on it.

```dockerfile
FROM alpine

RUN apk add --update redis

CMD ["redis-server"]
```

to build our image (convert above commands to an existing docker image) we run:

```shell
docker image build .
```

with the above command, we're telling docker that find a `Dockerfile` in existing directory. If you check your existing images throgh `docker images ls` you will find it on the top of list and you may run it as container:

```shell
docker run my_image_id
```

The pattern we used to build the image is very common, look at that 3 commands `FROM` `RUN` `CMD`. You may have more different commands in your docker file, but those commands are the most common ones and they almost repeat in every docker file.

## Behind the scenes

- At the first line docker looks for the **alpine** image, because of `FROM` directive, we're building an image based on **alpine** image on our local machine, because it doesn't exist, docker pulls it from docker hub
- The next step is about `RUN` directive, when the docker reads the seconds line, it create a tempurary container from the **alpine** image and runs the `apk add --update redis` on it. In some versions of the docker client you even can see the temp container's id. Then the docker takes snapshot of the temp container's file system as  an image and removes the temp container. So now, we have an new image that contains **redis** on it's FS.
- The third step is `CMD` directive. As  I said in the previous step, we got an image from `RUN` (step ||) command, so docker will create a contrainer from that image and run the `redis-server` command on the new temp container. Our building instruction finished, So again, docker saves the temp container's snapshop as an image and it's the final result of `docker build` operation.

### Docker puts stuff on cache

As you may seen, docker uses cache as much as possible, If you add a line on above `Dockerfile` you can see cache usage clearly. Let's add `RUN apk add --update gcc` so our dockerfile looks like this:

```dockerfile
FROM alpine

RUN apk add --update redis

RUN apk add --update gcc

CMD ["redis-server"]
```

Because the apline image exists on our local machine, docker won't download it from docker hub. In the next step docker uses cache because at the previous example, docker already built it. The next line is installing gcc that is new step. Docker is gonna run it completly, the reason obviously is gcc installation doesn't exist in previouse dockerfile.

What if we swap the lines of installation redis and gcc? As you could guess, docker won't use cache and it''s because when it wanted to install gcc the previouse image built on previouse dockerfile was created from  redis image, and now docker should create it from the alpine image, so it's an other story. As you seen, the order of instructions is so important to docker.

### Building an image through a running container

As you saw in **behind the scenee** section, at the every step, docker built on new image based on the previouse step's container, so it should be possible for us to do this with our arbitary containers. Good new! It is. Let's run the alpine image:

```dockerfile
docker run -it alpine sh
```

As you know, docker runs a container from the alpine image and we have the control of it's interactive shell. Let's do something, something that change our container's file system.

```dockerfile
apk add --update redis
```

Now our container has change on it's file system (redis added). How we can create a new image based on our container? Obviously our image must contains redis. Let's do this:

```dockerfile
docker commit -c 'CMD ["redis-server"]' CONTAINER_ID
```

Now you have a new image and you could push it on your docker hub if you want.

## Let's build something real

We gained some experiences in images and containers from previous sections. To recap them we gonna work on a simple Nodejs application and run it with docker.

To install dependencies we create a `package.json` fille in the root of repository with below content:

```json
{
    "dependencies": {
        "express": "*"
    },
    "scripts": {
        "start": "node index.js"
    }
}

```

Imagaine a simple express application with a single endpoint:

```js
const express = require('express');

const app = express();

app.get('/', (req, res) => {
    res.send('Hi there');
});

app.listen(8080, () => {
    console.log('Huray! Listening on 8080.');
});
```

Now lets create a docker file for the purpose. There is a point, in the previouse sections we used the `alpine` as base image. What if we try to use it for current project? The point is `alpine` doesn't ship with `nodejs` in default. We may want to use it, but we have to install nodejs using `RUN` directive and maybe it has some other dependencies. The easier way is using the [official node image](https://hub.docker.com/_/node). Obviously node image has been installed node and npm. The official image has a lot of tags they ship with several versions of node, different operations systems (including alpine) and everything.

It's time to create a `Dockerfile`

```dockerfile
FROM node:12-alpine

RUN npm install

CMD ["node", "start"]
```

How it sounds? there is some issues. when we want to build the image using `docker build .` we should get errors.

Our repository located on our local machine's file system, so there is nothing inside the container. when docker runs `npm install`, npm looks for a `package.json` file inside the container and it doesn't exist there. The solution is using `COPY` directive. It helps us to copy our files from host machine into the container. let's change file to below version:

```dockerfile
FROM node12-alpine

COPY ./ ./

RUN npm install

CMD ["npm", "start"]
```

 Note that `COPY ./ ./` says that: Hey docker! copy everything from current directory on my computer into the container. Now, problem solved and we can build our image:

```sh
docker build -t johndoe/my-app
docker run johndoe/my-app
```

Let's check it using `curl localhost:8080`. Does it repond? Probably no. The point `8080` port is exposed just **inside** container, the curl command sends request to host machine that doesn't serve anything on `8080`. What can we do? 

When we're building the image, there is an option to binding the ports. One side host machine another side container. Let's try something new:

```sh
docker run -p 8080:8080 johndoe/my-app
```

`-p` flag (means publish) tells the docker that: mount `8080` of host machine to `8080` of container. It works, but with current dockerfile we are copying source code into `/` of the container which is not cool. It's better to conider a directory for our application, Let's say `/app` and modify the dockerfile.

```dockerfile
FROM node12-alpine

COPY ./ /app

WORKDIR /app

RUN npm install

CMD ["npm", "start"]
```

There's an extra `WORKDIR` directive. The purpose is when we copy source code into `/app` docker won't switch working directory automatically. It still runs commands from `/`, so we should go to `/app` before run `npm install` and it's because our `package.json` is there now.

As final note, What if we change the source code? Probably we should build the image again. So docker should build everything again. We may help docker to use it's cache optimizer. We probably want to change the `index.js` file, the change rate of `package.json` file is fewer than the `index.js`. Because we are copying everything from host machine, docker detects there is a change, so it won't use cache for this step. Let's change our copy strategy.

```dockerfile
FROM node:12-alpine

WORKDIR /app

COPY ./package.json /app

RUN npm install

COPY ./ /app

CMD ["npm", "start"]

```

 No docker is able to use the cache in rebuilds, beacause the first four steps have no change, so docker build our image in faster way