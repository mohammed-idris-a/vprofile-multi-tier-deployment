# VProfile — Multi-Tier Application Deployment

A hands-on infrastructure project focused on deploying a multi-tier Java web application in a local virtualized environment using Vagrant, VirtualBox, and Linux services.

The project explores virtual machine provisioning, infrastructure configuration, service deployment, inter-service communication, and troubleshooting.

## Project Overview

VProfile is a Java-based web application supported by multiple infrastructure services.

The objective of this project is to provision the required infrastructure locally, configure the individual services, deploy the application, and verify that the complete application stack functions correctly.

The initial implementation follows a guided Udemy course, with environment compatibility adjustments, implementation records, troubleshooting, and additional improvements documented as the project progresses.

## Application Architecture

| Component | Technology | Responsibility |
|---|---|---|
| Web server / Load balancer | Nginx | Receives HTTP requests and routes traffic to the application |
| Application server | Apache Tomcat | Hosts the Java web application |
| Database | MariaDB | Stores application data |
| Caching | Memcached | Caches frequently accessed data |
| Message broker | RabbitMQ | Demonstrates message-broker infrastructure |

RabbitMQ is included as a demonstration component in the initial course project and is not functionally integrated into the application workflow.

## Infrastructure Architecture

The application stack is provisioned across five virtual machines.

| Virtual machine | Service | Responsibility |
|---|---|---|
| db01 | Database | Database service |
| mc01 | Memcached | Caching service |
| rmq01 | RabbitMQ | Message broker |
| app01 | Tomcat | Java application hosting |
| web01 | Nginx | Web server and load balancing |

### Provisioning Tools

- Oracle VirtualBox — Virtualization platform
- Vagrant — Virtual machine provisioning
- Vagrant Host Manager — Automatic hostname-to-IP mapping
- Linux — Guest operating systems

## Deployment Workflow

The intended service configuration sequence is:

1. Database
2. Memcached
3. RabbitMQ
4. Tomcat
5. Nginx

The infrastructure is first provisioned using Vagrant. Once the virtual machines are available, the services are configured and validated individually before testing the application through the web server.

## Project Documentation

| Document | Description |
|---|---|
| [Architecture](docs/ARCHITECTURE.md) | Application and infrastructure architecture |
| [Deployment](docs/DEPLOYMENT.md) | Step-by-step deployment procedure |
| [Troubleshooting](docs/TROUBLESHOOTING.md) | Issues, investigation and resolutions |

## Learning Objectives

- Understand multi-tier application architecture.
- Provision multiple virtual machines using Vagrant.
- Configure and manage Linux infrastructure services.
- Understand communication between application tiers.
- Deploy and verify a Java web application.
- Troubleshoot infrastructure and service-level failures.
- Improve the reproducibility and reliability of the local environment.

## Troubleshooting

Deployment issues encountered during provisioning and configuration, along with their investigation and resolutions, are documented here:

[View Troubleshooting Documentation](docs/TROUBLESHOOTING.md)

## Source Attribution

This project follows the VProfile course by hkhcoder.

Original project: https://github.com/hkhcoder/vprofile-project

The original application source code and instructor-provided configuration files belong to their respective authors. I do not claim authorship or ownership of these materials.

The original course material is distinguished from my own configuration changes,  infrastructure provisioning, implementation work, troubleshooting, verification, and independently developed improvements.

## Project Status

**Core Deployment Completed and Verified**

Successfully provisioned the five-VM environment, configured the required infrastructure services, built and deployed the Java application to Tomcat, and accessed the application through Nginx.

Deployment procedures and troubleshooting experiences are documented in the project documentation.

