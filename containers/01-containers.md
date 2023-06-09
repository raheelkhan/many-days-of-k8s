The docker images consist of layers. Each line in a Dockerfile creates a new layer. One important point to note here is that use of `rm` command inside Dockerfile only creates a new layer where this `rm` command is run. The deleted file will still be part of the image its just that it will not be visible in the final image layer. So in order to make our docker image size optimal we must take care of this consideration.

As we know that docker caches the layers when creating images but this is only be beneficial if we carefully write out Dockerfiles. 

Consider this scenario

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY ./server.js  ./
COPY package.* / ./
RUN npm install
CMD ["npm", "start"]
```

The above Dockerfile is an example of bad practice. Because the source code in the above case `server.js` is likely to change more often than the `package.json` file. But because we have put the `COPY ./server.js ./` command before we copy `package.json` and `RUN npm install`. If we run `docker build .` every time we modify `server.js` file, the docker engine will detect this change and try to create this layer again instead of using a cached layer. This result in creation of all the subsequent layers instead of docker utilizing the cached layers. So we should write our Dockerfile in such a way that any thing that is more likely to change should come in the end of Dockerfile so that the cache is utilized correctly. The above Dockerfile can be written in a best practice way like

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.* / ./
RUN npm install
COPY ./server.js  ./
CMD ["npm", "start"]
```

### How to push docker image to Docker Registry
First we have to tag the image that we have build with our user name

`docker tag typescript:thin raheelwp/typescript:thin`

Now If I do `docker images` I can see the image tagged with my username

```
raheelwp/typescript           thin           99b8e6411459   23 minutes ago   180MB
```

Now I can do `docker login` give it my credentials and then run `docker push raheelwp/typescript:thin`

### Resource limit

Docker allows us to limit containers resources usage on the host machine by a feature of Linux kernel called `cgoups`.

We can limit a container in terms of memory , swap and cpu for example

```
docker run -d --name kuard -p 8080:8080 --memory 200m --memory-swap 1G --cpu-shares 1024 gcr.io/kuar-demo/kuard-amd64:blue
```

> swap memory is a file or  partition on the disk which is used by kernel if the computer is running short of memory

> we can use different flags for cpu limiations such as `--cpu-period`, `--cpus`, `--cpu-quota` etc. More on this can be read at [Docker Documentation](https://docs.docker.com/config/containers/resource_constraints/)

