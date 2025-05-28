# Shepherd (Traefik)

> WORK IN PROGRESS

Builds given git repos periodically and automatically deploys them to a Linux box running
[Traefik](https://traefik.io).
Serves as a homebrew "replacement" for Heroku, to publish your own pet projects.
Built with off-the-shelf tools: Jenkins and Traefik.

How this works:

* All apps run as docker images.
* Traefik listens on https 443 (or http 80 in toy mode) and proxies requests to appropriate docker images.
* Jenkins rebuilds project Docker images and restarts docker containers automatically.
* Docker service [keeps the docker containers up-and-running](https://mvysny.github.io/vaadin-docker-service/)
* [Shepherd Web Admin](https://github.com/mvysny/shepherd-java-client) allows easy Shepherd administration via a browser.

In more details:
* You have a DNS domain, e.g. `foo.com`, pointing to your machine.
* The machine runs Traefik which [proxies requests](https://mvysny.github.io/2-vaadin-apps-1-traefik/) to appropriate docker images.
  * Apps are then published at `myapp23.foo.com`
  * Traefik also unwraps https to http and uses Let's Encrypt to obtain and update https certificates.
* Jenkins runs in Docker like all other hosted apps, at `jenkins-admin.foo.com`
  * Traefik routes to Jenkins like to any other app.
  * To isolate Jenkins from other apps for security reasons, Jenkins runs on its own private Docker network, `admin.int`
  * Needs to be accessible from outside, so that plugins can be upgraded; also we can't restart Jenkins when there are ongoing builds...
  * Jenkins simply runs `/opt/shepherd-traefik/shepherd-build PROJECTID` to rebuild the project and to start the docker container.
* Shepherd Web Admin runs in Docker as well, hosted at `admin.foo.com`
  * Also runs on private Docker network `admin.int`
* A https certificate for the wildcard DNS is obtained automatically by Traefik: [2 Vaadin apps 1 Traefik](https://mvysny.github.io/2-vaadin-apps-1-traefik/)
* Every project runs in docker container:
  * The main web service exposes port 8080 and has its own private network, which keeps the apps separated.
  * The docker container is named `shepherd_PROJECTID`; the docker image is named `shepherd/PROJECTID`; the docker network is named `PROJECTID.shepherd`
  * TODO Postgres Service

> Note: Original Shepherd used Kubernetes, however Kubernetes uses a lot of CPU for its upkeep,
> and makes the system much more complicated than it needs to be. See the [previous Vaadin Shepherd](https://github.com/mvysny/shepherd)
> if you're interested.

## Minimum requirements:

* A VM with 8-16 GB of RAM; both x86-64 and arm64 works. Ideally with a public ipv4 address.
* Fairly new Linux distribution, ideally Debian-based. Perfect is Ubuntu latest LTS, at least 24.04, so that
  all necessary apps are available in apt repository.
  * Docker 24 or higher, in order to be able to use docker build caches.
* A DNS domain, with the ipv4 "A" DNS record pointing to your VM.
  * Two "A" records are needed, with name "@" and "*" so that wildcard domains will work as well.

# Installation

ssh into the machine as root & update all packages & reboot.

Clone this project on the target machine:
```bash
$ cd /opt && git clone https://github.com/mvysny/shepherd-traefik && cd shepherd-traefik
```

To install Shepherd-Traefik, simply run `sudo ./install` script. This script is intended
to be run on Ubuntu 24.04+; if you have something else, see the sources of the `install`
script and run the commands accordingly.

## https

Configure Traefik to use https via Let's Encrypt in DNS wildcard mode:
[2 Vaadin Apps 1 Traefik](https://mvysny.github.io/2-vaadin-apps-1-traefik/).

# Adding Your Project To Shepherd

> Tip: The [shepherd-java](https://github.com/mvysny/shepherd-java-client) is a far easier way to add your projects.
> This way also works, but is more low-level, requires you to write docker compose yaml config files and fiddle with Jenkins,
> and is more error-prone. shepherd-cli calls this project anyway, but its project config file is far simpler.

Shepherd expects the following from your project:

1. It must have `Dockerfile` at the root of its git repo.
2. The Docker image can be built via the `docker build --no-cache -t test/xyz:latest .` command;
   The image can be run via `docker run --rm -ti -p8080:8080 test/xyz` command.
3. You can now register the project to Shepherd. **Continue to the "Adding a project" chapter** below.

Generally, all you need is to place an appropriate `Dockerfile` to the root of your project's git repository.
See the following projects for examples:

1. Gradle+Embedded Jetty packaged as zip: [vaadin-boot-example-gradle](https://github.com/mvysny/vaadin-boot-example-gradle),
   [vaadin14-boot-example-gradle](https://github.com/mvysny/vaadin14-boot-example-gradle),
   [karibu-helloworld-application](https://github.com/mvysny/karibu-helloworld-application),
   [beverage-buddy-vok](https://github.com/mvysny/beverage-buddy-vok),
   [vok-security-demo](https://github.com/mvysny/vok-security-demo)
2. Maven+Embedded Jetty packaged as zip: [vaadin-boot-example-maven](https://github.com/mvysny/vaadin-boot-example-maven)
3. Maven+Spring Boot packaged as executable jar: [Liukuri](https://github.com/vesanieminen/ElectricityCostDashboard),
   [my-hilla-app](https://github.com/mvysny/my-hilla-app).

## Maven+WAR

For Maven+war project, please use the following `Dockerfile`:

```dockerfile
# 1. Build the image with: docker build --no-cache -t test/xyz:latest .
# 2. Run the image with: docker run --rm -ti -p8080:8080 test/xyz

# The "Build" stage. Copies the entire project into the container, into the /app/ folder, and builds it.
FROM maven:3.9.1-eclipse-temurin-17 AS BUILD
COPY . /app/
WORKDIR /app/
RUN mvn -C clean test package -Pproduction
# At this point, we have the app WAR file in
# at /app/target/*.war
RUN mv /app/target/*.war /app/target/ROOT.war

# The "Run" stage. Start with a clean image, and copy over just the app itself, omitting gradle, npm and any intermediate build files.
FROM tomcat:10-jre17
COPY --from=BUILD /app/target/ROOT.war /usr/local/tomcat/webapps/
EXPOSE 8080
```

If your app fails to start, you can get the container logs by running:
```bash
$ docker exec -ti CONTAINER_ID /bin/bash
$ cat /usr/local/tomcat/logs/localhost.*
```

## Vaadin Addons

Vaadin addons are set up in a bit of an anti-pattern way:

* They package as jar, with the addon code only;
* The routes are packaged in the `src/test/` folder;
* They run via `mvn jetty:run`

The downside is that there's no support for production, and running via `mvn jetty:run`
requires Maven+Maven Repo+node_modules to be packaged in the docker image, increasing its size.

The solution is to:

* Replace Jetty with [Vaadin Boot](https://github.com/mvysny/vaadin-boot), adding Vaadin Boot as a test-scope dependency
* Creating a `Main.java` in `src/test/java/` which runs the app in Vaadin Boot.
* Creating Dockerfile which is able to build and run such app.

See [#16](https://github.com/mvysny/shepherd/issues/16) for more details; example project can be found at [parttio/breeze-theme](https://github.com/parttio/breeze-theme).

For addons that run via test-scoped Spring Boot, see the `Dockerfile` of the [parttio/parity-theme](https://github.com/parttio/parity-theme) example project.

