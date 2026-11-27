# OMERO server with Docker Compose

This repository contains Docker Compose examples for running OMERO.server, OMERO.web, and PostgreSQL. It is based on the [OME OMERO Docker example](https://github.com/ome/docker-example-omero) and includes an optional custom Bio-Formats image build.

## QBiC and I3D:bio context

QBiC uses this repository as a Docker Compose prototyping and validation workspace for OMERO. It is used to stand up representative servers, test patches, version updates, plugins and integrations, and validate candidate production configurations before rollout. Test production configurations in isolated development or staging deployments; this repository is not, by itself, the complete production service definition.

This work supports QBiC's contribution to **I3D:bio**, especially WP2's effort to build interoperable bioimage and omics RDM for spatial-omics use cases such as spaXio. The target infrastructure connects specialist systems while keeping data at their authoritative locations:

- **QBiC Data Manager software (pending ITSS confirmation):** broader Project → Experiment → Sample → Measurement context and links to related omics and other data resources; it is not the image-pixel store.
- **OMERO:** image-centric Project → Dataset → Image organization, image pixels or references to managed/external pixels, technical metadata, visualization, and image-related annotations/results.
- **Workflows and storage:** validated processing and metadata links across systems. SpatialData and OME-Zarr/OME-NGFF are candidates for selected spatial workflows, not a requirement to convert every assay or centralize every file in OMERO.

A Measurement in the Data Manager software (pending ITSS confirmation) and an OMERO Dataset are related but distinct records. A cross-system reference should retain the OMERO instance, object type, and object ID, with a web URL as a user-facing link.

The working service model separates internal collaborative curation, read-only publication-ready access, and a playground for technical testing. Curated data moves through QC/FAIR-readiness review before publication-ready access and indexing in the Data Manager software (pending ITSS confirmation); the playground remains outside that data lifecycle. This repository most directly supports the playground and pre-production validation. These are working architecture assumptions to confirm with service owners. See the [modernization roadmap](docs/modernization-roadmap.md).

## Workflows: Nextflow, nf-core, and Nidavellir

QBiC's workflow direction is **Nextflow**, using **nf-core tools and conventions**. [Nidavellir](https://github.com/qbic-pipelines/nidavellir/tree/dev) is the user-facing orchestration layer for FAIR-oriented bioimage training, inference, and data storage. Its development README follows the nf-core template and module conventions.

Nidavellir's documented OMERO companion is [OMERO Bifrost](https://github.com/luiskuhn/omero-bifrost/tree/forward-cycle), intended to provide query, push, and pull operations between Nextflow/nf-core workflows and one or more OMERO servers. The Nidavellir `dev` README says its current upload module still invokes the OMERO CLI directly; Bifrost-backed modules and constellation-aware workflow channels are planned. Treat this as an integration target, not as an already enabled capability of this Compose repository.

The OMERO deployments prototyped here should therefore be validated as workflow endpoints. For each candidate server update, qualify the tested OMERO CLI/Bifrost version and verify endpoint reachability, authentication and group permissions, image/metadata retrieval, output upload, and preservation of server-qualified object IDs and workflow provenance. Supply credentials securely at runtime; keep them out of samplesheets, pipeline parameters committed to Git, and output artifacts. Repeat these checks when changing the server, web stack, plugins, or storage configuration.

## Current configuration

The Compose files on `dev` are legacy examples; their pins are not a recommendation for a new production deployment.

| Component | `docker-compose.yaml` | `custom-bio-formats/docker-compose.yaml` |
| --- | --- | --- |
| PostgreSQL | `postgres:11` | `postgres:11` |
| OMERO.server | `openmicroscopy/omero-server:5.6.7` | Built from `custom-bio-formats/server/Dockerfile` |
| OMERO.web | `openmicroscopy/omero-web-standalone:5.21.0` | `openmicroscopy/omero-web-standalone:5.21.0` |

See the [modernization roadmap](docs/modernization-roadmap.md) for the version assessment, compatibility notes, and staged upgrade plan.

## Choose a Compose file

- **Standard setup:** use the root `docker-compose.yaml`. It uses the official OMERO.server and OMERO.web images.
- **Custom Bio-Formats build:** use `custom-bio-formats/docker-compose.yaml` only when you have confirmed that you need its custom Bio-Formats version. The Dockerfile is based on OMERO.server 5.6.7 and downloads a moving upstream POM during the build, so review and qualify it before relying on it.

Do not run both Compose projects against the same data directories at the same time.

## Start the standard setup

Requirements: Docker Engine with the Docker Compose v2 plugin, and a host with the directories used for persistent data.

The Compose file binds persistent data to `/mnt/omero/database` and `/mnt/omero/omeroserver` on the Docker host. Create those directories, then start the services:

```sh
sudo mkdir -p /mnt/omero/database /mnt/omero/omeroserver
docker compose config
docker compose up -d
docker compose ps
```

Open OMERO.web at `http://<server-host>:4080`. To follow service logs:

```sh
docker compose logs -f database omeroserver omeroweb
```

Run these commands from the repository root. For the custom Bio-Formats variant, change to `custom-bio-formats/` and build the server image before starting it:

```sh
cd custom-bio-formats
docker compose build omeroserver
docker compose up -d
```

## Services, ports, and data

| Service | Host port | Persistent data |
| --- | --- | --- |
| PostgreSQL | Not published | `/mnt/omero/database` |
| OMERO.server | `4063`, `4064` | `/mnt/omero/omeroserver` |
| OMERO.web | `4080` | None configured |

The database and OMERO binary repository are bind-mounted from the host. Back up both, along with the OMERO configuration and any custom scripts, and verify a restore before upgrading. See the [OME backup and restore guide](https://omero.readthedocs.io/en/stable/sysadmins/server-backup-and-restore.html).

## Deployment and upgrade notes

- The Compose files contain the example password `omero_passwd`. Replace it before deployment; keep credentials out of source control and use your deployment's secret-management mechanism.
- Ports `4063`, `4064`, and `4080` are published on the host. Restrict access with your firewall and provide HTTPS through your institution's reverse proxy before exposing web access beyond a trusted network.
- Do not upgrade PostgreSQL by changing the image tag while reusing its existing data directory. A PostgreSQL major-version upgrade needs a planned migration, backups, and validation.
- Pin and test compatible OMERO.server, OMERO.web, and database versions together. Review the roadmap before updating these Compose files.

## Related resources

- [Modernization roadmap](docs/modernization-roadmap.md)
- [OME announcements](https://www.openmicroscopy.org/announcements/)
- [OME Docker images](https://hub.docker.com/u/openmicroscopy)
- [OME OMERO Docker example](https://github.com/ome/docker-example-omero)
- [OMERO.server Docker image](https://github.com/ome/omero-server-docker/)
- [OMERO server environment variables](https://omero.readthedocs.io/en/stable/sysadmins/unix/server-installation.html#server-env)
- [OMERO server configuration](https://omero.readthedocs.io/en/stable/sysadmins/cli/config.html)
- [Bio-Formats OMERO customization](https://github.com/sbesson/bio-formats-omero/)
