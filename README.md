# Shepherd (Traefik)

Builds given git repos periodically and automatically deploys them to a Linux box running
[Traefik](https://traefik.io).
Serves as a homebrew "replacement" for Heroku, to publish your own pet projects.
Built with off-the-shelf tools: Kubernetes and Jenkins.

See the [previous Vaadin Shepherd](https://github.com/mvysny/shepherd).

How this works:

* Jenkins monitors projects and rebuilds them as Docker images automatically upon change
* Docker service [keeps the docker containers up-and-running](https://mvysny.github.io/vaadin-docker-service/)
* Apps are published at http://projectid.v-herd2.eu
* Traefik [proxies requests](https://mvysny.github.io/2-vaadin-apps-1-traefik/) to appropriate docker images.
* A wildcard DNS is obtained for `*.v-herd2.eu` automatically by Traefik.

Original Shepherd used Kubernetes, however Kubernetes uses a lot of CPU for its upkeep,
and makes the system much more complicated than it needs to be.

TODO work in progress
