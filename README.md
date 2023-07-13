# omero-docker
OMERO server setup using docker-compose.

## Description

This repository contains a solution to setup an OMERO server using docker-compose. It allows the control/definition of the Bio-formats library for the server. This solution is based on this post: [`How to control Bio-formats version of OMERO server using docker-compose`](https://forum.image.sc/t/how-to-control-bio-formats-version-of-omero-server-using-docker-compose/82935). The docker-compose `yml` files are based on [`OME/docker-example-omero`](https://github.com/ome/docker-example-omero). It also contains a copy of the `pom.xml` from [`sbesson/bio-formats-omero`](https://github.com/sbesson/bio-formats-omero/) under `./custom-bio-formats/bio-formats-omero`.

## Resources

- [OME announcements](https://www.openmicroscopy.org/announcements/)
- [OME Docker site](https://hub.docker.com/u/openmicroscopy)
- [OMERO docker-compose](https://github.com/ome/docker-example-omero)
- [OMERO server docker](https://github.com/ome/omero-server-docker/)
- [OMERO env. variables](https://omero.readthedocs.io/en/stable/sysadmins/unix/server-installation.html#server-env)
- [OMERO server config](https://omero.readthedocs.io/en/stable/sysadmins/cli/config.html)
- [omero.properties](https://github.com/ome/openmicroscopy/blob/develop/etc/omero.properties)
- [Bio-formats OMERO](https://github.com/sbesson/bio-formats-omero/)
- [OMERO docker build example](https://github.com/ome/docker-example-omero-websockets/blob/master/docker-compose.yml)
- [HAProxy config example](https://github.com/IDR/deployment/blob/master/ansible/templates/haproxy.cfg.j2)
