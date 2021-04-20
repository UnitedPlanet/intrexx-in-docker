# Intrexx in Docker

__ATTENTION:__ Please refer to the Docker Hub page, for a list of available image names.

Intrexx in Docker creates an ensemble of contains to host an Intrexx instance plus a portal. It builds on the basic idea that the portal is not part of the image. Instead, the image is just an Intrexx runtime (stripped heavily from a default intrexx installation) plus the blank portal template. Docker Compose is used as a deployment tool.

There are four use-cases:

1. If the deployment is started for the first time, a blank portal is created.
2. Alternatively, a zip file containing a portal export may be provided. That portal is then imported, instead of creating a blank portal.
3. If the portal volume contains a portal of the current version, it is started.
4. If the portal is not yet of the current version, it is patched and then started.

Case 4 is not implemented yet.

# Prerequisites

In order to deploy Intrexx as a Docker container, we use [docker-compose](https://docs.docker.com/compose/). This is done, because Intrexx can not be run as a standalone container but needs a database and a search server to work correctly. Optionally, an nginx server can be useful as reverse proxy. Managing the deployment as a whole is enabled by docker-compose.

In order to follow these guidelines, you need to have Docker and Docker Compose configured on your machine. Please follow the official installation guidelines:

- [Docker installation](https://docs.docker.com/get-docker/)
- [Docker Compose installation](https://docs.docker.com/compose/install/)

# Usage

## Create new (blank) portal

This is the most simple use-case. No changes are needed. Just run

```bash
docker-compose up -d
```

This will create three containers: database, solr server and portal. The portal directory will be created in a named Docker volume. This may be used to backup data but also persists the portal between upgrades.

## Create new portal from export

Many users might already have a portal (export) that they wish to deploy in Docker. To achieve that, the portal must be provided in zip format.

As described above, the portal directory (/opt/intrexx/org) is a named volume in the Docker Compose deployment. If a portal zip is provided within this volume, it is used as template for portal creation, instead of the blank portal. This workflow however requires for the volume to be a mounted directory from the host system instead.

To mount the the directory on the host system, the docker-compose.yml must be adjusted as follows:

```yml
# Case 1 (portal is provided from mounted host directory)

intrexx:
  ...
  environment:
    PORTAL_ZIP_NAME: my_portal.zip
  ...
  volumes:
    - /portal/dir/on/local/system:/opt/intrexx/org
```

If this is not wanted, an additional volume can be configured, containing only the portal zip and mounted into the Intrexx container. In this case, the (absolute) path to mount point within the container must be provided as environment variable (PORTAL_ZIP_MNTPT). The docker-compose.yml must then be adjusted as follows:

```yml
# Case 2 (portal is provided from additional volume)

intrexx:
  ...
  environment:
    PORTAL_ZIP_NAME: my_portal.zip
    PORTAL_ZIP_MNTPT: /tmp/templates
  ...
  volumes:
    - portal-data:/opt/intrexx/org
    - /portal/dir/on/local/system:/tmp/templates

```

In both cases, the portal can again be started by running

```bash
docker-compose up -d
```

## Destroy deployment

To completely destroy a deployment, use

```bash
docker-compose down -v
```
__Attention:__ This will also remove the Docker volumes so no data is left! To stop the volumes without removing them (so they can be restarted), use

```bash
docker-compose down
```

## Upgrade deployment

To update to a newer version of Intrexx, optionally configure the desired version in the docker-compose.yml or leave as is, to use the latest available version. Then run these commands:

```bash
docker-compose down
docker-compose pull
docker-compose up
```

By doing so, the Intrexx container will be destroyed, the image updated and started again as a new Intrexx container. Meanwhile the volume persists. On startup the already present portal is detected, patched to the new Intrexx version and started.
Please be aware that the startup may take extended time because of the required portal patch.

## Configure nginx as reverse proxy

If you want to use nginx as a frontend webserver and thereby enable SSL encryption, the procedure depends on whether the portal is directly mounted from the host machine or contained in an extra volume.

In both cases you follow these steps:

- In the `docker-compose.yml` enable the commented lines for the nginx service. Please be aware of the correct indentation so that `nginx` is recognized as an additional container. Also adjust the `PORTAL_BASE_URL` environment variable.
- Within the directory `resource/nginx/ssl/<Your server's DNS>` provide the needed certificate files and a DH param file for your server.
- Within the files `docker-compose.yml` and `resource/nginx/conf.d/default.conf` replace all occurrences of `example.unitedplanet.de` with your desired DNS.
- If you changed the portal name (by modifying the EVN Var PORTAL_NAME) also adjust the htmlroot path in `resource/nginx/conf.d/default.conf`.

If the portal is contained in the separate portal-data volume (case 2 above), you are done now. If you mounted the portal from a local directory instead (case 1 above), adjust the volume definition in the nginx service in `docker-compose.yml` as well:

```yml

nginx:
  ...
  volumes:
  ...
    - /portal/dir/on/local/system:/opt/intrexx/org
```

If your deployment is already running, you may now use the following commands to start your NGINX frontend:

```bash
docker-compose up --no-start nginx
docker-compose start nginx
```

If the deployment is not already running, use `docker-compose up -d` instead as above and the NGINX service will be started alongside your deployment.

# Tags

The Intrexx images are tagged with the semantic version of the release. The semantic version contains 3 parts:

- Major version
- Minor version
- Patch level (online update number)

So, for example, the semantic version of Intrexx 21.03 OU05 would be 10.0.5.

Additionally for every major version, there is a specific latest tag, e.g. `21.03-latest`. This may be used to automatically receive the most recent patches but to avoid major version upgrades.

Please refer to the [Docker Hub page](https://hub.docker.com/r/unitedplanet/intrexx) for a complete list of available tags.

# FAQ

## Is this deployment scalable?

No. If you need horizontal scaling, please use the [Cloud Playbooks](https://github.com/UnitedPlanet/intrexx-cloud-playbooks).

## Can I have multiple portals?

No. This deployment runs exactly one Intrexx portal. For additional portals, please make additional deployments.

## How can I connect with the portal manager?

As there is only one portal per deployment, a supervisor is not necessary and therefore not included. As a consequence, a connection to the portal can not be established via a supervisor (i.e. port 7960) as usual. Instead, it must must be connected [directly to the portal](https://onlinehelp.unitedplanet.com/intrexx/10000/de-de/Content/Onlinehelp-Intrexx/helpfiles/de.uplanet.lucy.client.lucymanager.portaldirect.ConnectPage.html). On a local machine (e.g. for development purposes) the portal can be reached either through the host address 0.0.0.0 (alternatively the local IP) and the default port 8101.

## Where do I find the logs?

A combined log from all containers can be obtained from Docker compose directly via

```bash
docker-compose logs -f
```

Alternatively, the separate logs are available directly from the containers via

```bash
docker-compose logs -f [intrexx|db|solr|nginx]
# e.g.: docker-compose logs -f intrexx
```

## Can I use another database type?

Currently, only PostgreSQL is supported out of the box. While it is possible to modify deployment and image to use another database, no support is provided by United Planet.

## Can I use this in production?

Generally yes. However, since this is the first version of Intrexx that is deployable in Docker, we currently can not guarantee that there will be no issues. However, we offer full support when encountering problems caused from our side in the deployment of Intrexx in Docker. We continue to work steadily on improving our scripts in order to enable a deployment that runs as smooth as possible.
