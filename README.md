# Intrexx in Docker
The basic idea is, that the portal is not part of the image. Instead, the image is just a intrexx runtime (stripped heavily from default intrexx installations) and the blank portal template. docker-compose is used as a deployment tool.
There are 4 use-cases:

1. If the deployment is started the first time, a blank portal is created.
2. Alternatively, a zip file containing a portal export may be provided. That portal is then imported, instead of creating a blank portal.
3. If within the portal volume a portal of the current version is found, it is started.
4. If the portal is not yet the current version, it is patched and then started.

Case 4 is not implemented yet.

# Prerequisites
In order to deploy intrexx as a docker container, we use [docker-compose](https://docs.docker.com/compose/). This is done, because intrexx can not be run as a standalone container but needs a database and a search server to work correctly. Optionally a nginx server can be useful as reverse proxy.
Managing the deployment as a whole is enabled by docker-compose.

In order to follow these guidelines, you need to have Docker and docker-compose configured on your machine. Please follow the official installation guidelines:
- [docker installation](https://docs.docker.com/get-docker/)
- [docker-compose installation](https://docs.docker.com/compose/install/)

# Usage
## Create new (blank) portal
This is the most simple use-case. No changes are needed. Just run
```bash
docker-compose up -d
```

This will create 3 containers: database, solr server and portal.
The portal directory will be created in a named docker volume. This may be used to backup data but also persists the portal between upgrades.

## Create new portal from export
Many users might already have a portal (export) that they wish to deploy in docker. To archieve that, the portal must be provided in zip format.

As described above, the portal directory (/opt/intrexx/org) is a named volume in the docker-compose deployment. If within this volume a portal zip is provided, it is used as template for portal creation, instead of the blank portal. This workflow however requires for the volume to be a mounted directory from the host system instead. If this is not wanted, an additional volume can be configured, containing only the portal zip and mounted into the intrexx container. In this case, the (absolute) path to mountpoint within the container must be provided as environment variable (PORTAL_ZIP_MNTPT).

For each of the cases, the docker-compose.yml must be adjusted as showed here:
```yml
# Case 1
environment:
  PORTAL_ZIP_NAME: my_portal.zip
...
volumes:
  - /portal/dir/on/local/system:/opt/intrexx/org

# Case 2
environment:
  PORTAL_ZIP_NAME: my_portal.zip
  PORTAL_ZIP_MNTPT: /tmp/templates
...
volumes:
  - portal:/opt/intrexx/org
  - /my/portal/template/dir:/tmp/templates
```

Starting the deployment is then archieved, by using:
```bash
docker-compose up -d
```

## Destroy deployment
To completely destroy a deployment, use
```bash
docker-compose down -v
```
__Attention:__ This will also remove the docker volumes so no data is left!

## Upgrade deployment
To update to a newer version of intrexx, optionally configure the desired version in the docker-compose.yml or leave as is, to use the latest available version. Then run these commands:

```bash
docker-compose down
docker-compose pull
docker-compose up
```

By doing so, the intrexx container will be destroyed, the image updated and started again as a new intrexx container. Meanwhile the volume persists. On startup the already present portal is detected, patched to the new intrexx version and started.
Please be aware, that the startup may take extended time, because of the portal patch needed.

## Configure nginx as reverse proxy
If you want to use nginx as a frontend webserver and thereby enable SSL encryption, you can do so by following these steps:
- In the `docker-compose.yml` enable the commented lines for the nginx service
- Within the directory `resource/nginx/ssl` provide the needed certificate files and a DH param file for your server
- Within the files `docker-compose.yml` and `resource/nginx/conf.d/default.conf` replace all occurences of `example.unitedplanet.de` with your desired DNS
- If you changed the portal name (by modifying the EVN Var PORTAL_NAME) also adjust the htmlroot path in `resource/nginx/conf.d/default.conf`
- If you mounted a local directory instead of using the portal-data volume, adjust the volume definition in the nginx service in `docker-compose.yml` as well

If your deployment is already runngin, you may now use the following commands, to start your nginx frontend:
```bash
docker-compose create nginx
docker-compose start nginx
```
If not, just run the above described `up` commands and the nginx service will be started with your deployment.

# Tags
The intrexx images are tagged with the semantic Version of the Release. The semantic Version contains 3 Parts:
- Major Version
- Minor Version
- Patch Level (OU Number)

For example, the semantic Version of intrexx 21.03 OU05 would be 10.0.5.

Additionally, for every major version, there is a specific latest tag. e.g. `21.03-latest`. This may be used, to always receive the newest patches but no major version upgrades automatically.

# FAQ
## Is this deployment scalable?
No. If you need horizontal scaling, please use the [Cloud Playbooks](https://github.com/UnitedPlanet/intrexx-cloud-playbooks).

## Can I have multiple portals?
No. This deployment runs exactly 1 Intrexx Portal. For additional portals, please make additional deployments.

## How can I connect with the portal manager?
As there is only 1 portal per deployment, the supervisor is not needed and therefore not running. This means, that the manager can not connect to the supervisor (Port 7960) as usual but must be connected [directly to the portal](https://onlinehelp.unitedplanet.com/intrexx/10000/de-de/Content/Onlinehelp-Intrexx/helpfiles/de.uplanet.lucy.client.lucymanager.portaldirect.ConnectPage.html) instead.

## Can I use another database type?
Currently only PostgreSQL is supported out of the box. While it is possible to modify deployment and image to use another database, no support by United Planet is provided.

## Can I use this in production?
Generally yes. But as this is the first version of Intrexx that is deployable in Docker, we can not guarantee that there will be no issues currently. However, we offer full support for possible problems caused by our side in the deployment of Intrexx in Docker. We continue to work steadily on improving our deployment scripts in order to enable a deployment that runs as smooth as possible.
