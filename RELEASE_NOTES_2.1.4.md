> Description
STINGARv2.1 Release
Contains updates to all docker images with latest base images and bug fixes.

### Upgrade Steps

* [ACTION REQUIRED]
* Requires new docker-compose.yml & stingar.env files to install new images.

### Breaking Changes

* New Elasticsearch database install - all locally stored honeypot attack data will be lost (see user guide for backup/restore options) but honeypot IP & Auth Keys retained.

### New Features

* New UI implementation with enhanced dashboard and honeypot management features.
* Updated user documentation
* Updated data retention policy (90day hot, 180day warm, 365day auto-delete) for Elasticsearch
* Updated base images for all containers for SBOM compliance
* Support for ARM64 & Intel64 vm hardware architectures
* Honeypot data enrichment with geopoint, city, country of attacker origin

### Bug Fixes

*
*

### Other Changes

* new sign-in on initial install on clean vm - requests admin account password
