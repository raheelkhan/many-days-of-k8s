I wanted to try how to optimize the docker image size. After all it takes space in the node where it will run if we talk about Kubernetes. It also increases the overall deploy time because the larget the image size, the more time it will take to be pulled. For this purpose I prepared a little demo

You can see that I have a Dockerfile in this folder that creates a nodejs helloworld program. I assumed that I will be using Typescript for my development. Which means if I have to build this image, The image must have typescript compiler `tsc` present. But we know that it is a devDependency and ideally it should not be a part of image that we are going to deploy. devDependencies are only used in development or during build time.

The first image I created results in size 

```
typescript                    fat            c99469ee8067   About an hour ago   1.05GB
```

Then I use docker multistage build feature, and I also used alpine based image because now I don't need to do any fancy thing. All I need in my final image is the node runtime and only libraries that are actually required for code execution. The resulting size is as follows

```
typescript                    thin           99b8e6411459   6 seconds ago       180MB
```