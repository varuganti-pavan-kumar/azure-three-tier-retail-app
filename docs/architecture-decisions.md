# Architecture Decisions

This document records the main architecture choices for the HA Retail Azure project, why they were made, and their current limitations.

**Last updated:** 7 November 2026

## Decision 1: Use a three-tier application design

**Decision:** Separate the storefront, application API, and database into distinct tiers.

**Implementation:**
- Web tier: Nginx on the web VM in South India.
- Application tier: Python API on the app VM in South India.
- Data tier: Azure Database for PostgreSQL Flexible Server in UK South.

**Reason:** Separating these responsibilities makes the system easier to understand, secure, troubleshoot, and extend.

**Trade-off:** Separate VMs and cross-region database connectivity add infrastructure and networking complexity and can increase cost and latency.

**Status:** Deployed and tested over the private network.

## Decision 2: Use Nginx as the web server and reverse proxy

**Decision:** Serve the storefront through Nginx and forward API requests to the app VM.

**Implementation:**
- Storefront files are served from `/var/www/html`.
- `/api/` is forwarded to `10.0.3.4:8080`.
- `/health` is forwarded to the API health endpoint.

**Reason:** The browser can use a single web origin for the storefront and API paths. The app VM does not need to serve the static website.

**Trade-off:** Nginx configuration and the web VM become important parts of the request path.

**Status:** Deployed and tested. Homepage, health, and product API requests returned HTTP 200 through Nginx.

## Decision 3: Keep the application VMs in private subnets

**Decision:** Do not assign public IP addresses to the web and app VMs.

**Implementation:**
- Web subnet: `10.0.2.0/24`.
- App subnet: `10.0.3.0/24`.
- Web VM private IP: `10.0.2.4`.
- App VM private IP: `10.0.3.4`.
- The app VM accepts TCP port `8080` from the web subnet.

**Reason:** Private addressing reduces direct exposure of the virtual machines to the public internet.

**Trade-off:** Access to the VM-hosted website requires an authorized private network path. Public internet access is not currently configured.

**Status:** Implemented. Neither VM has a public IP.

## Decision 4: Use PostgreSQL Flexible Server in a separate region and VNet

**Decision:** Place the database in UK South, separate from the application VNet in South India.

**Implementation:**
- PostgreSQL Flexible Server: `ha-retail-southindia-pg`.
- PostgreSQL version: 18.
- Database: `retaildb`.
- PostgreSQL VNet: `ha-retail-southindia-postgresql-vnet`.
- PostgreSQL VNet address space: `10.1.0.0/16`.
- Delegated subnet: `10.1.1.0/24`.
- Observed private IP: `10.1.1.4`.
- VNet peering and private DNS provide private connectivity and name resolution.

**Reason:** This demonstrates private connectivity between separately addressed Azure virtual networks and regions.

**Trade-off:** Cross-region connectivity adds latency and networking dependencies. A separate region does not automatically provide application high availability or disaster recovery.

**Status:** Deployed and tested. PostgreSQL public access is disabled. The server uses a delegated subnet, not a Private Endpoint.

## Decision 5: Use managed identity and Key Vault for database credentials

**Decision:** Do not store the PostgreSQL password in application source code or the server-side environment file.

**Implementation:**
- The app VM uses a system-assigned managed identity.
- The identity has the `Key Vault Secrets User` role at the vault scope.
- The API retrieves `retailapp-db-password` from Key Vault.
- The API connects to PostgreSQL using SSL.

**Reason:** Managed identity avoids putting a separate Azure credential in the application configuration. Key Vault provides a central place to manage the database password.

**Trade-off:** The API needs permission and network connectivity to Key Vault during startup. If secret retrieval fails, API startup fails.

**Network detail:** Key Vault public network access is restricted using a default-deny firewall and an app-subnet service endpoint/VNet rule. This is not a Private Endpoint configuration.

**Status:** Implemented and tested.

## Decision 6: Use a dedicated PostgreSQL application login

**Decision:** The API connects using the `retailapp` database login rather than a general administrator login.

**Implementation:**
- Database: `retaildb`.
- Schema: `app`.
- Application table: `app.products`.
- The application login has the required schema and database permissions.

**Reason:** A dedicated login separates application access from administrative database access.

**Trade-off:** The login's permissions must be maintained as the application changes.

**Status:** Implemented and tested.

## Decision 7: Store sample product records in PostgreSQL

**Decision:** The API reads product records from the database instead of returning a hardcoded response.

**Implementation:**
- The API creates `app.products` if it does not exist.
- It inserts three sample products using `ON CONFLICT DO NOTHING`.
- The `/api/products` endpoint queries the table.

**Reason:** This demonstrates an actual application-to-database data path while keeping the initial demo easy to reproduce.

**Trade-off:** The sample data is not a real retail catalogue, and startup performs basic database initialization.

**Status:** Implemented and tested. The API returned three sample records with `source` set to `postgresql`.

## Decision 8: Use systemd to manage the API process

**Decision:** Run the Python API as a Linux systemd service.

**Implementation:**
- Service: `ha-retail-api.service`.
- Service user: `ha-retail-api`.
- Port: `8080`.
- The service restarts after an unexpected failure.

**Reason:** systemd provides service startup, restart handling, and status inspection without requiring a terminal session to remain open.

**Trade-off:** systemd restart handling does not provide multiple instances or automatic failover.

**Status:** Production service active and enabled. The temporary test service on port `8081` is stopped and disabled.

## Decision 9: Keep separate Terraform configuration directories

**Decision:** Retain both `terraform/` and `terraform-southindia/` in the repository.

**Reason:** The repository contains more than one infrastructure configuration, allowing the project to retain its earlier and region-specific infrastructure definitions.

**Trade-off:** Separate configurations can cause confusion if the wrong directory or state is used.

**Operational rule:** Inspect the Terraform files, variables, backend configuration, and state before running `plan`, `apply`, or `destroy`. Never assume the directories share state.

**Status:** Both directories are present in the repository. Their exact deployment relationship should be reviewed before future infrastructure changes.

## Decision 10: Keep public ingress and high availability as future work

**Decision:** Do not claim that the current private VM deployment is publicly reachable or highly available.

**Current implementation:**
- No public IP was found for either VM.
- No Application Gateway was found in the resource group.
- The tested Nginx storefront is private.
- The application has one web VM and one API VM.
- No automatic failover has been demonstrated.

**Reason:** This documents the actual deployed system instead of describing planned architecture as completed work.

**Trade-off:** The current deployment is not publicly accessible through the VM-hosted storefront and has single-instance failure points.

**Future work:**
- Multiple API instances.
- A load balancer and health probes.
- Redundant web and application infrastructure.
- Database backup and restore testing.
- Monitoring, alerting, and disaster recovery tests.

**Status:** High availability and public ingress are not implemented.

## Summary

The project demonstrates a private three-tier Azure application with a working Nginx-to-Python-to-PostgreSQL request path, managed identity, Key Vault secret retrieval, private networking, and Terraform configuration.

It should currently be described as a **private three-tier Azure application demo**, not as a completed high-availability or public-facing production system.