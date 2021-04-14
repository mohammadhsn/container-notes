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



