# OMERO Docker modernization: state of the art and roadmap

**Review date:** 2026-11-26
**Starting point:** `dev` at `48d568ba9e41879d6862122d7f39455d9d02ecb6`
**Status:** Planning note for review; no production configuration or database has been changed.

At review time, `origin/dev` and `origin/main` resolve to the same commit. The work is still based on `dev` as requested.

## QBiC and I3D:bio context

QBiC uses this repository to prototype OMERO deployments with Docker Compose and to validate technical changes: server and web updates, patches, plugins, integrations, and candidate production configurations. Configuration intended for production should first be exercised in an isolated development or staging deployment. The repository is a technical prototyping and validation resource, not the complete production service definition.

This work supports QBiC's contribution to **I3D:bio**, particularly WP2's interoperable RDM architecture for bioimage and omics data and selected spatial-omics use cases such as spaXio. The target is a distributed infrastructure that preserves authoritative data locations and links specialist systems:

- **QBiC Data Manager software (pending ITSS confirmation):** project, experiment, sample, and measurement context, plus references to related omics and other data resources.
- **OMERO:** image-centric Project → Dataset → Image management, technical image metadata, visualization, and image annotations/results. It may manage pixels in its binary repository or reference external pixels where a validated workflow supports that model.
- **Workflows and storage:** validated metadata, processing, and transfer pathways. SpatialData and OME-Zarr/OME-NGFF are candidates for selected spatial workflows; they do not imply a universal format conversion or a single repository for all modalities.

A Measurement in the Data Manager software (pending ITSS confirmation) and an OMERO Dataset are distinct records. Cross-system references should preserve system/instance, object type, and object ID; a webclient URL is useful for navigation but should not be the only identifier. Keep high-dimensional molecular resources in their appropriate systems rather than treating OMERO as the repository for all omics data.

The working QBiC service model has three distinct OMERO roles: internal collaborative curation, read-only publication-ready access, and a playground for testing. Curated bioimage data can move through QC and FAIR-readiness review to publication-ready OMERO access and indexing in the Data Manager software (pending ITSS confirmation). The playground is outside that data lifecycle and is reserved for disposable or approved non-production fixtures. This repository most directly supports that playground and the pre-production validation of production candidate configurations. Confirm instance topology, ownership, and promotion procedures with service owners; these are working architecture assumptions rather than an approved deployment specification.

## Nextflow, nf-core, and Nidavellir integration

QBiC's workflow direction is **Nextflow**, using **nf-core tools and conventions**. [Nidavellir](https://github.com/qbic-pipelines/nidavellir/tree/dev) is the user-facing orchestration layer for FAIR-oriented bioimage training, inference, and data storage; its development README follows the nf-core template and module conventions.

Nidavellir's documented OMERO companion is [OMERO Bifrost](https://github.com/luiskuhn/omero-bifrost/tree/forward-cycle), designed for query, push, and pull operations between Nextflow/nf-core workflows and single or federated OMERO servers. The Nidavellir `dev` README states that its current upload module still calls the OMERO CLI directly, while Bifrost-backed modules and constellation-aware workflow channels remain planned. These capabilities are an integration target, not a feature already provided by this Compose repository.

Qualify the OMERO deployment as a workflow endpoint alongside server upgrades. Record the compatible OMERO CLI/Bifrost revision and validate network reachability, authentication and group permissions, image and metadata retrieval, output upload, and provenance. Preserve server-qualified object identities (server or profile plus object type and local ID) across workflow handoffs. Supply credentials securely at runtime; do not put them in samplesheets or committed pipeline configuration. Repeat integration checks after changes to OMERO, plugins, networking, identity, or storage.

## Executive summary

The `dev` Compose file is a 2023-era setup: PostgreSQL 11, OMERO.server 5.6.7, and OMERO.web 5.21.0. PostgreSQL 11 has been unsupported upstream since November 2023. The current stable OMERO server release documented by OME is 5.6.18; it bundles Bio-Formats 8.5.0 and is documented as tested with OMERO.py 5.22.1 and OMERO.web 5.31.0. OMERO.web 5.33.4 was released on 2026-11-18. PostgreSQL 19 was released on 2026-11-19, but OME's published OMERO 5.6 dependency matrix only covers PostgreSQL through 16. Therefore, PostgreSQL 16 is the most defensible production target from the currently published compatibility evidence; evaluate 17–19 separately before adopting them.

Recommended target to validate in staging:

| Component | Current `dev` pin | Candidate | Rationale / gate |
|---|---:|---:|---|
| PostgreSQL | 11 | 16.x, latest minor | Supported by OMERO's published 5.6 matrix through Nov 2028. PostgreSQL 19 is upstream-current but not in that matrix. |
| OMERO.server | 5.6.7 | 5.6.18 | Current documented 5.6 release; includes Bio-Formats 8.5.0. |
| OMERO.web | 5.21.0 | 5.33.4 | Current release found on review date; first validate against the server using the documented 5.31.0 pairing, then qualify 5.33.4. |

This is an upgrade across a major PostgreSQL boundary and a large OMERO/Bio-Formats gap. Treat it as a staged migration with a verified rollback, not as a routine image-tag edit.

## What the branch currently does

`docker-compose.yaml` runs a PostgreSQL 11 container, `openmicroscopy/omero-server:5.6.7`, and `openmicroscopy/omero-web-standalone:5.21.0`. Credentials are literal defaults in Compose (`omero_passwd`); ports 4063, 4064, and 4080 are published on the host. The database and `/OMERO` data store are bound to `/mnt/omero/...` host paths.

The `custom-bio-formats` variant builds on server 5.6.7 and copies Bio-Formats dependencies into the server using a 2023 copy of `bio-formats-omero`. It downloads a moving `master` POM, uses old Maven dependency-plugin configuration and HTTP Maven repositories, and builds with an unpinned Maven installation. It is not a safe drop-in base for a 5.6.18 server. First establish whether a required microscope format actually needs a custom Bio-Formats build. Prefer the official server image and its bundled Bio-Formats unless a format-specific test proves otherwise; if customization is required, pin source revision, dependency versions, artifact repositories and checksums, then build and test against the exact server release.

The repo README points to OME's Docker examples and official images, which should become the deployment source of truth. The Compose `version: "3"` field is obsolete in current Docker Compose and can be removed during implementation. Keep the Compose change separate from the database migration plan so configuration cleanup does not obscure migration failures.

## Compatibility and platform decisions

### OMERO.server, OMERO.py, and web

OME's 5.6.18 history entry reports tests with OMERO.py 5.22.1 and OMERO.web 5.31.0; use that combination as the documented compatibility baseline. The newer web release 5.33.4 should be tested as a second candidate in the staging environment, including the web plugins currently supplied by `omero-web-standalone` and any local customizations. Do not equate “latest tag exists” with “this whole deployment combination was tested.”

Use explicit image version tags, and preferably record image digests for deployed releases. Avoid floating `latest` tags. Confirm that the corresponding official Docker image tags have been published before updating Compose; release numbers in the Python package repositories and the Docker registry do not always become available at the same time.

Bio-Formats changed from the 6.12.0-era custom build in this repository to 8.5.0 in OMERO.server 5.6.18. OME documents that this change invalidates existing Bio-Formats Memoizer cache files. Plan cache regeneration and assess any importer or reader behavior that depended on the old custom build.

### PostgreSQL

PostgreSQL 11 is end-of-life. OMERO's 5.6 support table marks PostgreSQL 15 as recommended and 16 as supported; PostgreSQL upstream supports 16 through November 2028. PostgreSQL 19 is the upstream-current release on this review date, but the OMERO table does not list it. Select 16 for the first production migration unless OME publishes a newer explicit compatibility statement or local staging demonstrates otherwise. Keep its minor version current.

PostgreSQL major versions use distinct data directories. Do not simply change `postgres:11` to `postgres:16` while reusing the existing database volume. Take a consistent database dump and the OMERO binary repository backup, restore into a new PostgreSQL 16 data volume, run the OMERO database upgrade procedure, and validate before cutover. Preserve the old database and application data as a rollback set until the new instance passes acceptance.

### OME-Zarr / OME-NGFF

OME-Zarr is a released, evolving format rather than a single OMERO server switch. The current OME-NGFF specification is version 0.6. The official `omero-cli-zarr` extension provides command-line import/export for OMERO Images, Plates and ROIs; its in-place import path depends on `omero-zarr-pixel-buffer`, and import documentation limits source locations to a public object store or filesystem accessible to the OMERO server. The current extension documentation describes output formats through 0.5, so OME-Zarr 0.6 must be qualified instead of assumed.

An additional S3-compatible store is a reasonable target architecture for large OME-Zarr datasets. In the in-place model, object chunks stay in S3; OMERO creates Image/Plate records and retains an `ExternalInfo` pointer to the corresponding Zarr group. The pixel-buffer component reads the chunks when OMERO needs pixels. This avoids copying a second full pixel set into `/OMERO`, while OMERO remains the access and metadata layer. It does mean the Zarr objects, OMERO database records, and their relationship must be operated as one data service.

There is an important private-bucket integration caveat: `omero-cli-zarr` currently documents import only from public object stores or server-accessible filesystems. The separate pixel-buffer documentation describes AWS and S3-compatible storage, authenticated reads using a server credentials profile or cloud instance role, and anonymous reads for public stores. Therefore private S3-compatible storage may be readable by the pixel buffer while the stock importer still cannot perform the initial import. Confirm this end-to-end before selecting the storage product; a custom registration workflow or importer enhancement may be needed. Do not make research data public just to satisfy an importer limitation.

For protected data, use a private bucket/container and give the OMERO service identity only the minimum read/list permissions it needs for the designated prefixes. Keep write permission with the ingest/pipeline identity, deliver credentials through the deployment's secret or workload-identity mechanism, and avoid embedding keys in Compose, image layers, or `ExternalInfo` URLs. Validate that OMERO group permissions gate pixel reads even though the server-side object-store credential is shared; do not assume OMERO ACLs automatically become per-user S3 ACLs. Use TLS, immutable/versioned object prefixes, object-store-side retention/versioning and backup, and stable endpoint/bucket/key paths. Have the ingest workflow validate the Zarr before publishing it and register it only after upload is complete.

Suggested lifecycle:

```mermaid
flowchart LR
    A[Acquisition / processing] --> B[Validate OME-Zarr]
    B --> C[Write immutable object prefix]
    C --> D[Register in OMERO in place]
    D --> E[OMERO metadata, groups and annotations]
    D --> F[Pixel reads via Zarr pixel buffer]
    F --> G[S3-compatible object store]
    E --> H[Data Manager software (pending ITSS confirmation) links to OMERO IDs]
```

Separate the intended use cases:

1. **Export package:** test workflow-based OME-Zarr export for representative 2D, 3D, time-series and HCS examples; validate metadata, labels/ROIs, chunking, pyramid levels, and downstream readers.
2. **External / in-place data:** test the full upload → validation → registration → rendered-pixel read path with the exact `omero-cli-zarr` and pixel-buffer versions, storage endpoint, and authentication model. Test private S3-compatible storage explicitly; public-store examples do not demonstrate private-bucket support. Define whether OMERO only references external pixels or imports a managed copy, and document ACLs, lifecycle, deletion, backup, restore, and endpoint/key stability.
3. **Web viewing:** evaluate a compatible OMERO.web viewer extension separately. A web viewer/plugin does not itself establish server-side import, pixel management, or access-control behavior.

For in-place objects, treat the object-store path as part of the OMERO record's identity. Avoid overwriting chunks beneath a registered key: publish a new immutable prefix for changed data and create a new OMERO record or an explicit version/provenance link. Define a de-registration procedure so removing an OMERO record cannot accidentally delete shared objects, and define the reverse retention policy for objects whose OMERO record was removed. Test endpoint outages and credential expiry so operators know how failures appear to users.

Do not treat conversion of every spatial assay to OME-Zarr as a goal. Define a small supported profile by modality and workflow, preserve acquisition originals, and retain metadata/provenance linking the derived Zarr dataset to the source and processing run. This is especially important for SpatialData-based multi-modal objects, which have their own model and should not be collapsed into “OME-Zarr” without an explicit mapping.

### Authentication and ORCID

OMERO's documented built-in external authentication mechanism is LDAP. ORCID supports OAuth 2.0 authorization-code flow and OpenID Connect for researcher identity. These facts do not make ORCID a supported OMERO identity provider by default: OMERO core does not document generic OIDC login. A third-party `omero-oauth` web app exists, but its README says it implements alternate login using an OMERO administrative account to create users/sessions and points issue support to the incubator project. Treat it as an unqualified security-sensitive integration, not a production-ready turnkey feature.

Recommended direction:

- Use the institution's managed identity provider (typically LDAP or an institutional OIDC/SAML broker) for account authentication and lifecycle, subject to a tested OMERO mapping and authorization model.
- Use ORCID primarily as a verified researcher identifier linked to the OMERO/person record in the Data Manager software (pending ITSS confirmation) or publication metadata.
- If direct “Sign in with ORCID” remains a requirement, run a separate proof of concept. Review code and maintenance status, token/session handling, account linking and duplicate accounts, user/group provisioning, logout/revocation, MFA expectations, and admin credential scope. Require HTTPS end to end and do not expose an OMERO administrator password to a web plugin unless the threat model and controls have been reviewed.

## Upgrade roadmap

### Phase 0 — inventory and acceptance criteria

Record the *running* versions and deployment facts; the `dev` pins may not equal the deployed server. Inventory database size/version, image-store size and mount, backups and restore time, external file references, custom Bio-Formats readers, OMERO scripts/web plugins, LDAP/auth settings, ports/proxy/TLS, user/group model, and the formats plus instruments used. Agree on maintenance window, RPO/RTO, rollback owner, and acceptance test set.

### Phase 1 — backup and secure the staging baseline

Prove restoration of both the PostgreSQL dump and the complete `/OMERO` binary repository, plus OMERO configuration and any custom scripts. OME identifies the database, binary data store, and configuration as the key backup sources and cautions that live backups may not be consistent. Build a staging stack on separate volumes; remove committed default passwords, keep secrets out of Git, restrict direct host exposure, and place TLS/reverse-proxy handling at the institutional edge. Confirm firewall needs for desktop clients before closing 4064.

### Phase 2 — staged database and OMERO upgrade

Use a cloned/sanitized production backup. Restore into a new PostgreSQL 16 volume, then follow the OMERO 5.6 server upgrade guide for the database schema and server. Start with OMERO.server 5.6.18 and the documented OMERO.py/web compatibility set (web 5.31.0) to isolate the main upgrade. Run server diagnostics, login/group/permission checks, image rendering, import/export, scripts/plugins, and the team's representative microscope-format tests. Regenerate invalidated Bio-Formats memoizer data as needed.

Once that is stable, test OMERO.web 5.33.4 as a separate change. Verify plugin compatibility, web login, rendering/downloads, reverse-proxy headers and static assets. Pin the selected image tags and digests in Compose only after the matrix is recorded.

### Phase 3 — OME-Zarr pilot

Select a few representative datasets and a target version (include the current 0.6 specification in evaluation, while recognizing the current importer/pixel-buffer material specifically describes 0.4 and 0.5 support). Stand up a private S3-compatible pilot bucket with a separate ingest writer identity and OMERO read-only service identity. Benchmark upload and validation, registration, random tile/plane reads, object request volume/latency, viewer behavior, labels/ROIs, HCS hierarchy, ACL enforcement, credential rotation, endpoint outages, and restore from object-store backup. Test one public reference dataset only as a functional baseline; never use public access for private study data. Decide whether canonical pixels stay in OMERO's managed repository or in the governed object store, and write down the stable path, metadata/provenance link, deletion, and recovery policies.

### Phase 4 — ORCID identity pilot

First document institution identity federation and confirm whether the requirement is institutional single sign-on, ORCID identity collection, or both. Keep those functions separate. If direct ORCID login is still needed, run a security-reviewed, isolated proof of concept with non-production accounts. Do not roll it into the database/server upgrade.

### Phase 5 — production cutover and operations

Schedule downtime; quiesce imports and OMERO writes; take final consistent backups; restore/migrate into the new database volume; start pinned server/web images; run smoke and acceptance checks; then redirect traffic. Keep the old volumes and images intact through the rollback window. After acceptance, document routine minor patching, image digest updates, backup verification, restore drills, storage monitoring, CVE review, and a quarterly compatibility review. Track PostgreSQL major upgrades separately from OMERO application upgrades.

## Pre-implementation decisions / required site facts

- What versions and image digests are currently running in production, and how large are the DB and managed repository?
- Which formats specifically required the custom Bio-Formats build? Which importer plugins, scripts, and OMERO.web extensions are in use?
- Is OME-Zarr needed for import, external in-place browsing, export, or all three? Which AWS or S3-compatible service, private-bucket IAM/credentials model, access policy, data-retention/immutability policy, and OME-NGFF versions are expected?
- Does “ORCID authentication” mean login with ORCID, verified ORCID collection, or institutional SSO plus ORCID linking?
- Which network clients need direct access to Ice ports, and what proxy/TLS service is already operated?
- When was a full database + binary repository restore last proven, and what downtime is available?

## Primary sources

- [OMERO Docker Compose branch (`dev`)](https://github.com/qbicsoftware/omero-docker/tree/dev) and [official OMERO Docker examples](https://github.com/ome/docker-example-omero)
- [OMERO 5.6 release history](https://omero.readthedocs.io/en/stable/users/history.html), [version requirements](https://omero.readthedocs.io/en/v5.6.15/sysadmins/version-requirements.html), [upgrade guide](https://omero.readthedocs.io/en/stable/sysadmins/server-upgrade.html), and [backup and restore guide](https://omero.readthedocs.io/en/stable/sysadmins/server-backup-and-restore.html)
- [OMERO.web releases](https://github.com/ome/omero-web/releases) and [official OMERO.web Docker image guidance](https://github.com/ome/omero-web-docker)
- [PostgreSQL versioning policy](https://www.postgresql.org/support/versioning/) and [PostgreSQL 19 release notes](https://www.postgresql.org/docs/current/release.html)
- [OME-NGFF released specifications](https://ngff.openmicroscopy.org/specifications/index.html), [OME `omero-cli-zarr`](https://github.com/ome/omero-cli-zarr), [Glencoe `omero-zarr-pixel-buffer`](https://github.com/glencoesoftware/omero-zarr-pixel-buffer), and [Zarr 3 support issue describing the pixel-buffer's 0.4/0.5 and authenticated S3 test target](https://github.com/glencoesoftware/omero-zarr-pixel-buffer/issues/44)
- [ORCID integration and API FAQ](https://info.orcid.org/documentation/integration-and-api-faq/), [ORCID sign-in guidance](https://info.orcid.org/documentation/integration-guide/orcid-oauth-sign-in-guidelines/), [OMERO LDAP authentication](https://omero.readthedocs.io/en/stable/sysadmins/server-ldap.html), and [third-party `omero-oauth` README](https://github.com/microscopepony/omero-oauth)

Version and support statements above are snapshots from 2026-11-26. Recheck release tags, image registry availability, and OME's compatibility matrix immediately before implementation.
