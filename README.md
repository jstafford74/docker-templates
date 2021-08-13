# Docker Mastery for Node.js Projects From a Docker Captain

This repo is for use in my Udemy Course https://www.bretfisher.com/docker-mastery-for-nodejs

Feel free to create issues or PRs if you find a problem with the repo. Please use the Udemy course for Q&A and course support.

## Docker compose

https://docs.docker.com/compose/

- Compose YAML v2 vs v3
- v2 focus: single node dev/test
- v3 focus: multi-node orchestration

- If not using swarm/Kubernetes, stick to v2
  https://docs.docker.com/compose/compose-file/compose-versioning/
  https://docs.docker.com/compose/compose-file/
  https://docs.docker.com/compose/reference/
  https://wiki.alpinelinux.org/wiki/Alpine_Linux_package_management

- build: just buildrebuild images
- stop: just stop containers dont delete
- ps: list "services"
- logs: same as CLI
- exec: same as CLI

## Node.js FROM images

- COPY, not ADD(unless you know why)
- npm/yarn install during build
- CMD node, not npm
  - npm requires another application to run
  - not as literal in Dockerfiles
  - doesn't work well as an init or PID 1 process
- WORKDIR not RUN mkdir
  - unless need chown

## FROM Base Image Guidelines

- Stick to even numbered major releases
- Don't use :latest tag
- Start with Debian if migrating
- Move to Alpine later
  - Small base image with minimal tools
- Don't use :slim
- Don't use :onbuild

## When to use Alpine Images

- Alpine is small and "sec focused"
- Debian/Ubuntu are smaller now ~ 85 - 100 MB
- ~100MB space savings isnt significant
- Alpine has its own issues
  - different than the default
- Alpine CVE(common vulnerability scanner) scanning fails
  https://kubedex.com/follow-up-container-scanning-comparison/
  https://www.youtube.com/watch?v=e2pAkcqYCG8

- Enterprises may require CentOS or Ubuntu/Debian

## Make a CentOS Node Image

- Install Node in the official CentOS
- Copy Dockerfile lines from node:10
- Use ENV to specify node version

## Least Privilege: Using node User

- Official node images have a node user, but not enabled by default
- Do this after apt/apk and npm i -g
- Do this before npm i
- May cause permissions issues with write access
  - chmod & chown
- Change uder from root to node
  - USER node
- Set permissions on app dir
  - RUN mkdir app && chown -R node:node .
- Run a command as root in container

- Linux file commands
- Docker commands

## Making Images Efficiently

- Pick proper FROM
- Line order matters
- COPY twice: `package.json*` then `. .`
- One apt-get per Dockerfile
  - apt-get high up in file

## Node Process Management

- Best ways to start Node containers

  - No need for nodemon, forever, or pm2 on server
  - Use nodemon in dev for file watch later
  - Docker manages app start, stop, restart, healthcheck
  - Node multi-thread: Docker manages multiple 'replicas'
  - One npm/node problem: They don't listen for proper shutdown signal by default.

- Truth about PID1 Problem

  - PID 1 (Process Identifier) is the first process in a system (or container) AKA init
  - Init process in a container has 2 jobs:
    - reap zombie processes
    - pass signals to sub-processes
  - Zombie not a big Node issue
  - Proper shutdown

    - Docker uses Linux signals to stop app
      - SIGINT = ctrl+c
      - SIGTERM = docker-compose stop
      - SIGKILL = should not use
    - SIGINT/SIGTERM allow graceful stop
    - npm doesn't respond to SIGINT/SIGTERM
    - node doesn't respond by default, but can with code
    - Docker provides a init PID 1 replacement option
      https://docs.docker.com/engine/reference/run/#specify-an-init-process

    - Options:
      - Temp: Use --init to fix ctrl+c for now
      - Workaround: add `tini` to image
      - Production: app captures SIGINT for proper exit
    - Run any node app with --init to handle signals
      - `docker run --init -d nodeapp`
    - Add tini to Dockerfile
    - Production solution: Graceful shutdown

## Multi-stage Builds

    https://docs.docker.com/develop/develop-images/multistage-build/
    - Multiple FROM's with multiple results
    - Images can FROM each other
    - COPY files between images
    - Space & security benefits
    - Artifact only images

    https://medium.com/@tonistiigi/advanced-multi-stage-build-patterns-6f741b852fae

## 12 Fact App container happiness (Heroku creators)

- https://12factor.net/
- Use environment variables for config
  - https://12factor.net/config
- Store environment config in env vars
  - Anything that changes in dev, test or prod is a config and needs to pulled out of app - Log to stdout & stderr
  - https://12factor.net/logs
  - Should not route or transport logs to anything but stdout/stderr
  - `console.log()` works
  - Winston/Bunyan/Morgan: Use levels to control verbosity - Winston transport: "Console" - Pin all versions, even npm - Graceful exit SIGTERM/INIT - Create .dockerignore
  - Prevent bloat and uneeded files
    .git/
    node_modules/
    npm-debug
    docker-compose\*.yml

* Not needed but useful in image
  Dockerfile
  README.md

## Compose Project Tips

    - Use docker-compose for local dev
    - v2 & v3
        - v2 local environment
            * Only: depends_on & hardware specific
        - v3 multi-server environment
    - Study compose files and CLI features
    - Dont's:
        * Unecessary: 'alias' & 'container_name'
        * Legacy: 'expose' & 'links'
        * No setting defaults

### Bind-Mounting Code

    - usually `./`
    - Don't use host file paths
    - DOn't bind mount DB's

https://www.docker.com/blog/user-guided-caching-in-docker-for-mac/

### Startup Order & Depoendencies

- Dependency Awareness
  - depends_on: when "up X", start y first
  - Fixes name resolution issues with "can't resolve <service_name>"
  - Only for compose, not Orch
  - Compose YAML v2: works with healthchecjs like a "wait for script"
- Name resolution (DNS)
- Connection failure handling
  - restart: on-failure
    - Good: helps slow db startup & Node.js failing. Better: depends_on
    - Bad: could spike CPU with restart cycling
    - Solution: build connection timeout, buffer, and rettries in your apps.

### Healthchecks for depends_on

- depends_on: is only dependency control by default
- Add v2 healthchecks for true "wait_for"

### Making Microservices Easier

- Problem: many Http endpoints, many ports
- Solution: Nginx/HAProxy/Traefik for host header routing + wildcard localhost domain
  Problem: CORS failures in dev
  Solution: Proxy with \* header
  https://letsencrypt.org/docs/certificates-for-localhost/
  https://github.com/nginx-proxy/nginx-proxy
  https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/
  https://medium.com/@sumankpaul/use-nginx-proxy-and-dnsmasq-for-user-friendly-urls-during-local-development-a2ffebd8b05d
  https://traefik.io/
  https://github.com/docker-solr/docker-solr/issues/182
  https://github.com/nginx-proxy/nginx-proxy/issues/804

### vscode/typescript

- https://code.visualstudio.com/docs/nodejs/nodejs-debugging
- https://github.com/Microsoft/vscode-recipes/tree/master/
- https://github.com/TypeStrong/ts-node

### Building a Sweet Compose File

- Shrinking Compose Files and DRY YAML
  While not Node-specific, I thought I'd remind you of three significant features in docker-compose YAML that saves you time, prevents repetitive lines in each service (keeping it DRY), and create flexibility for large projects and teams.

### Dockerfile Documentation

https://docs.docker.com/engine/reference/builder/#label
https://docs.docker.com/config/labels-custom-metadata/
https://github.com/opencontainers/image-spec/blob/main/annotations.md

- Document every line that isn't obvious
- FROM stage, document why it's needed
- COPY = don't dopcument
- RUN = maybe document
- Add Labels
- RUN npm config list

#### Example Dockerfile Labels

- Environment variables
  Eventually, you'll need a compose file to be flexible and you'll learn that you can use environment variables inside the Compose file. Note, this is not related to the YAML object environment, which you want to send to the container on startup. With the notation of ${VARNAME}, you can have Compose resolve these values dynamically during the processing of that YAML file. The most common examples of when to use this are for setting the container image tag or published port. If your docker-compose.yml file looks like this:

version: '2'
services:
ghost:
image: ghost:${GHOST_VERSION}
...then you can control the image version used from the CLI like so:

GHOST_VERSION=2 docker-compose up

You can also set those variables in other ways: by storing them in a .env file, by setting them at the CLI with export, or even setting a default in the YAML itself with ${GHOST_VERSION:-2}. You can read more about variable substitution and various ways to set them in the Docker docs.

Templating
A relatively new and lesser-known feature is Extension Fields, which lets you define a block of text in Compose files that is reused throughout the file itself. This is mostly used when you need to set the same environment objects for a bunch of microservices, and you want to keep the file DRY (Don't Repeat Yourself). I recently used it to set all the same logging options for each service in a Compose file like so:

version: '3.4'

x-logging:
&my-logging
options:
max-size: '1m'
max-file: '5'

services:
ghost:
image: ghost
logging: *my-logging
nginx:
image: nginx
logging: *my-logging
You'll notice a new section starting with an x-, which is the template, that you can then name with a preceding & and call it from anywhere in your Compose file with \* and the name. Once you start to use microservices and have hundreds or more lines in your Compose file, this will likely save you considerable time and ensure consistency of options throughout. See more details in the Docker docs.

Control your Compose Command Scope
The docker-compose CLI controls one or more containers, volumes, networks, etc., within its scope. It uses two things to create that scope: the Compose YAML config file (it defaults to docker-compose.yml) and the project name (it defaults to the directory name holding the YAML config file). Normally you would start a project with a single docker-compose.yml file and execute commands like docker-compose up in the directory with that file, but there's a lot of flexibility here as complexity grows.

As things get more complex, you may have multiple YAML config files for different setups and want to control which one the CLI uses, like docker-compose -f custom-compose.yml up. This command ignores the default YAML file and only uses the one you specify with the -f option.

You can combine many Compose files in a layered override approach. Each one listed in the CLI will override the settings of the previous (processed left to right)—e.g., docker-compose -f docker-compose.yml -f docker-override.yml.

If you manually change the project name, you can use the same Compose file in multiple scopes so they don't "clash." Clashing happens when Compose tries to control a container that already has another one running with the same name. You likely have noticed that containers, networks, and other objects that Compose creates have a naming standard. The standard comprises three parts: projectname_servicename_index. We can change the projectname, which again, defaults to the directory name with a -p at the command line. So if we had a docker-compose.yml file like this:

version: '2'
services:
ghost:
image: ghost:${GHOST_VERSION}
ports: ${GHOST_PORT}:2368
Then we had it in a directory named app1 and we started the ghost app with inline environment variables like this:

app1> GHOST_VERSION=2 GHOST_PORT=8080 docker-compose up

We'd see a container running named this: app1_ghost_1

Now, if we want to run an older version of ghost side-by-side at the same time, we could do that with this same Compose file, as long as we change two things. First, we need to change the project name to ensure the container name will be different and not conflict with our first one. Second, we need to change the published port so they don't clash with any other running containers.

app1> GHOST_VERSION=1 GHOST_PORT=9090 docker-compose -p app2 up

If I check running containers with a docker container ls, I see:

app1_ghost_1 running ghost:2 on port 8080
app2_ghost_1 running ghost:1 on port 9090
Now you could pull up two browser windows and browse both 8080 and 9090 with two separate ghost versions (and databases) running side by side.

I hope these three features help your workflow, as they have done for many the teams I've worked with in creating more complex and flexible Compose files.
