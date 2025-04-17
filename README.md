# Shepherd (Traefik)

Builds given git repos periodically and automatically deploys them to a Linux box running
[Traefik](https://traefik.io).
Serves as a homebrew "replacement" for Heroku, to publish your own pet projects.
Built with off-the-shelf tools: Kubernetes and Traefik.

See the [previous Vaadin Shepherd](https://github.com/mvysny/shepherd).

How this works:

* Jenkins monitors projects and rebuilds them as Docker images automatically upon change
  * Installed as a deb, running natively in linux outside of docker.
    * Or alternatively running within docker itself! [docker-in-docker](https://mvysny.github.io/docker-in-docker/)
  * Listens as http on port TODO on `docker0` and `localhost` interfaces; Traefik will unwrap https for us.
  * Needs to be accessible from outside, so that plugins can be upgraded; also we can't restart Jenkins when there are ongoing builds...
    * This could be solved via a bash script perhaps?
* Shepherd Web Admin, listening as http on port TODO on the `docker0` interface; Traefik will unwrap https for us.
  * Or alternatively running within docker itself! [docker-in-docker](https://mvysny.github.io/docker-in-docker/)
* Docker service [keeps the docker containers up-and-running](https://mvysny.github.io/vaadin-docker-service/)
* Apps are published at `https://projectid.v-herd2.eu`
* Traefik [proxies requests](https://mvysny.github.io/2-vaadin-apps-1-traefik/) to appropriate docker images.
  * Tunnels `https://jenkins.admin.v-herd2.eu` to Jenkins: https://stackoverflow.com/a/43541732/377320
  * Tunnels `https://web.admin.v-herd2.eu` to Shepherd Admin
* A wildcard DNS is obtained for `*.v-herd2.eu` automatically by Traefik.
* [shepherd-java](https://github.com/mvysny/shepherd-java-client) should be modified to be able to control
  both the old shepherd and also the new one.
* Every project has a docker-compose yaml file named `projectid.yaml` in `/etc/shepherd/docker-compose/` folder
  * The main web service exposes port 8080 and is attached to a web network
  * If there's Postgres Service, a private network is generated as well, so that Vaadin sees Postgres

Original Shepherd used Kubernetes, however Kubernetes uses a lot of CPU for its upkeep,
and makes the system much more complicated than it needs to be.

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

## docker-compose YAML

Every project will have a docker-compose yaml file named `projectid.yaml` in `/etc/shepherd/docker-compose/` folder, such as:
```yaml
networks:
  web:
    external: true
services:
  vaadin:
    image: shepherd/karibu-helloworld-application:latest
    networks:
      - web
    ports:
      - "8080:8080"
    restart: always
    labels:
      - "traefik.http.routers.vaadin-boot-example-gradle.entrypoints=http"
      - "traefik.http.routers.vaadin-boot-example-gradle.rule=Host(`karibu-helloworld-application.v-herd2.eu`)"
```

If PostgreSQL service is enabled for the project, add the following to the YAML file: TODO

Now, run `docker-compose up -d -f /etc/shepherd/docker-compose/karibu-helloworld-application.yaml`.

You can further run the following commands to:
- TODO obtain logs
- TODO see whether the app is up

## Jenkins build

The Jenkins build script is as follows: TODO

# Shepherd Internals

Minimum requirements:

* A VM with 8-16 GB of RAM; both x86-64 and arm64 works. Ideally with a public ipv4 address.
* Fairly new Linux distribution, ideally Debian-based. Perfect is Ubuntu latest LTS, at least 24.04, so that
  all necessary apps are available in apt repository.
  * Docker 24 or higher, in order to be able to use docker build caches.
* A DNS domain, with the ipv4 "A" DNS record pointing to your VM.

## Installation

ssh into the machine as root & update all packages & reboot.

To install Shepherd-Traefik, simply run `sudo ./install` script.
Alternatively, go through the steps below.

### Docker

Install docker (make sure Docker is version 24 or higher):

```bash
$ sudo apt install docker.io docker-buildx docker-compose
$ usermod -aG docker 1000 # so that Jenkins has access to docker
$ reboot
```
Create a private docker network for administrative tools (Jenkins and Shepherd-Java):
```bash
$ docker network create admin.int
```

We'll use a lot of networks with docker - one network per project. The default limit of docker networks is 29
which means we can create and host 28 projects tops (minus 1 network allocated for admin stuff).
To be able to create more networks, edit `/etc/docker/daemon.json` and add:
```json
{
   "default-address-pools": [
        {
            "base":"172.17.0.0/12",
            "size":16
        },
        {
            "base":"192.168.0.0/16",
            "size":20
        },
        {
            "base":"10.99.0.0/16",
            "size":24
        }
    ]
}
```
Restart docker daemon:
```bash
$ systemctl restart docker.service
```

#### Docker Build Cache

We need docker-buildx to enable docker build cache (see [Docker buildx mount cache](https://mvysny.github.io/docker-build-cache/) for more details).
To take advantage of build caches, the project must contain this in their `Dockerfile`s:
```
RUN --mount=type=cache,target=/root/.gradle --mount=type=cache,target=/root/.vaadin ./gradlew clean build -Pvaadin.productionMode --no-daemon --info --stacktrace
```
Note the `--mount-type` arg.

We need separate build caches per project, otherwise we can't build projects in parallel since
they would overwrite their caches. See [#3](https://github.com/mvysny/shepherd/issues/3) for more details.

```bash
sudo mkdir -p /var/cache/shepherd/docker
sudo chgrp docker /var/cache/shepherd/docker
sudo chmod g+w /var/cache/shepherd/docker
```
Edit `/etc/docker/daemon.json` and enable containerd-snapshotter:
```json
{
  "features": {
    "containerd-snapshotter": true
  }
}
```
Restart docker daemon:
```bash
$ systemctl restart docker.service
```
Verify that the setting took effect:
```bash
$ docker info|grep driver-type
driver-type: io.containerd.snapshotter.v1
```

### Prepare The Filesystem

Create the necessary files on the filesystem:
```bash
$ sudo mkdir -p /var/opt/shepherd/jenkins_home
$ sudo chown 1000 /var/opt/shepherd/jenkins_home  # Jenkins runs as user 1000
```
Clone this project on the target machine:
```bash
$ cd /opt && git clone https://github.com/mvysny/shepherd-traefik && cd shepherd-traefik
```
Edit the `docker-compose.yaml` and replace `mydomain.me` with the DNS domain you own, and
you allocated for this project.

> Note: if you're just trying out Shepherd-Traefik then you should enable all lines labeled
> "debug" - it will disable https (simplifying the setup) and enable Traefik Web UI interface.

Now you're ready to start Shepherd:
```bash
$ docker-compose up
```
### https

Configure Traefik to use https via Let's Encrypt in DNS wildcard mode:
[2 Vaadin Apps 1 Traefik](https://mvysny.github.io/2-vaadin-apps-1-traefik/).

### Jenkins

Go to `https://jenkins.admin.mydomain.me` and configure Jenkins. Enter the generated admin password
(logged to stdout of your docker-compose), then:

* "Select plugins to install"
  * Disable "Folders", "Ant", all "Pipeline" plugins, "SSH Build Agents", "Matrix Auth Strategy", "PAM Auth", "LDAP", "Email Extension" and "Dark Theme".
* Create the `admin` user, with a good strong password.
* Go to *Manage Jenkins / System* and set `# of executors`: 2
