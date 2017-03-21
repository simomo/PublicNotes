# How to use docker to build a comfortable env for both dev & production

## The definition of "comfortable"
### No needs to deploy app env repeatly in dev and production env
I only need to take care of my code, my local dev env, and dockerfile, then production env can be generated easily.
### Local dev is fast & flexible
I want the process of "changing code -> refreshing -> show me the result" is fast, building image each time is unacceptable.
I can use any IDE/editor I like to develop, no limits.
### Involving CI system easily
Spur don't have QEs, so I will involve CI into its development process in (near) future, so it must be friendly for CI vendors.
## Basic idea
Three container, one for spur client (based on nginx), one for spur server (based on python), the last one is for DB (postgresql I guess?)
### About the docker file
The docker file will be under source code's root directory, so that I can use `COPY` command to copy source code into image when building one. 

> Reference 1: [Creating a Consistent Cross-platform Docker Development Environment](https://blog.codeship.com/cross-platform-docker-development-environment/)  
> Reference 2: [How to use --volume option with Docker Toolbox on Windows?](http://stackoverflow.com/a/42435077)  
> Reference 3: [How to Get Code into a Docker Container](http://blog.cloud66.com/how-to-get-code-into-a-docker-container/)  
> Reference 4: [Docker ADD vs VOLUME](http://stackoverflow.com/a/27735940/807695)  