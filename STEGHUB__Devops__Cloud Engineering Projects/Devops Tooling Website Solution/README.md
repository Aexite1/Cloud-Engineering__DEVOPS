# DevOps Tooling Website Solution

## Introduction

This repository contains my implementation of a DevOps tooling website solution — a three-tier application deployed across AWS infrastructure with shared NFS storage, a MySQL database, and multiple Apache/PHP web servers. The project documentation walks through the full build process, from provisioning the storage layer to logging into the deployed application.

## Repository Structure

```text
.
├── images/
├── SideStudy/
└── devops_tooling_website_solution.md
```

## Project Files and Folders

### `images/`

Screenshots captured during the implementation, covering each major step: LVM configuration on the NFS server, MySQL setup, web server provisioning, NFS mounts, Apache/PHP installation, application deployment, and the final working login page.

### `SideStudy/`

Additional notes and research gathered while working through the project — useful context that didn't fit directly into the main implementation guide.

### `devops_tooling_website_solution.md`

The core project document. It details the full three-tier deployment: configuring the NFS storage server with LVM, setting up the MySQL database layer, provisioning three RHEL web servers with Apache and PHP, mounting shared NFS storage, deploying the tooling application from GitHub, and connecting it all to the database.

## Project Overview

The goal was to build a working tooling website where multiple web servers serve the same content from shared NFS storage and connect to a single MySQL database. This makes the web tier stateless — servers can be replaced without losing data, since application files live on NFS and data lives in the database.

The implementation covers:

- **Storage layer** — RHEL EC2 instance with three LVM logical volumes (`lv-apps`, `lv-logs`, `lv-opt`) formatted as XFS and exported via NFS
- **Database layer** — Ubuntu EC2 instance running MySQL with a `tooling` database and a `webaccess` user restricted to the web server subnet
- **Web layer** — Three RHEL EC2 instances running Apache and PHP, each mounting `/var/www` and `/var/log/httpd` from the NFS server
- **Application** — The tooling application cloned from a forked GitHub repository, deployed to the shared NFS folder, and configured to connect to the MySQL database

## Technologies & Concepts

- AWS EC2, security groups, subnets
- Red Hat Enterprise Linux 10 and Ubuntu Linux
- LVM (physical volumes, volume groups, logical volumes)
- XFS filesystem
- NFS server and client configuration
- MySQL installation, user management, and remote access
- Apache HTTP Server, PHP-FPM, Remi repository
- SELinux configuration for Apache
- Git and GitHub forking/cloning
- Three-tier architecture and stateless web design

## Recommended Reading Order

1. Read `devops_tooling_website_solution.md` from start to finish to follow the implementation in order.
2. Cross-reference the screenshots in `images/` at each step — they show the actual terminal output and AWS console views.
3. Check `SideStudy/` for supplementary notes on concepts encountered during the build.

## Author

**Great Ojuolape**  
Computer Science Student | DevOps & Cloud Engineering Aspirant.

## License

This project is intended for educational and learning purposes.