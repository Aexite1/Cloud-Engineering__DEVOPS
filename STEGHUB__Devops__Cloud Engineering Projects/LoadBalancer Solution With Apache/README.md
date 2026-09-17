# Apache Load Balancer Solution

This repository documents my hands-on implementation of an **Apache Load Balancer** for a web application running across multiple web servers.

The project covers the preparation of the NFS and Web Servers, shared application storage, Apache configuration, load-balancing between backend servers, and testing the traffic distribution.

## Repository Structure

```text
.
├── images/
├── Sidestudy/
├── Load_Balancer_Solution_With_Apache.md
└── README.md
```

### 📁 images/

Contains the screenshots captured during the step-by-step implementation of the project.

The screenshots are referenced throughout the main documentation to provide a visual guide to the different stages of the setup.

### 📁 Sidestudy/

Contains additional study materials and notes related to the concepts involved in the project.

These materials were used to better understand the technologies and configuration steps during the implementation.

### 📄 Load_Balancer_Solution_With_Apache.md

This is the main project documentation.

It provides a detailed, step-by-step explanation of the implementation, including:

- NFS storage server configuration
- LVM and shared storage setup
- Web Server preparation
- NFS mounts
- Apache and PHP configuration
- Apache Load Balancer setup
- Backend server configuration
- Traffic distribution
- Load Balancer testing
- Local name resolution using `/etc/hosts`

The screenshots in the `images/` folder are used throughout the document to support the explanations.

## Project Overview

The goal of this project was to configure **Apache as a Load Balancer** on a separate Ubuntu EC2 instance and use it to distribute requests across multiple RHEL web servers.

The web servers use shared application storage through NFS, while Apache on the Load Balancer forwards incoming requests to the available backend servers.

### Technologies & Concepts

- AWS EC2
- Ubuntu
- Red Hat Enterprise Linux
- Apache HTTP Server
- Apache `mod_proxy_balancer`
- NFS
- LVM
- PHP
- MySQL
- Linux
- Git & GitHub

## Recommended Reading Order

1. Start with `Load_Balancer_Solution_With_Apache.md`.
2. Follow the screenshots in `images/` alongside the documentation.
3. Refer to `Sidestudy/` when additional background information is needed.

## Author

**Great Ojuolape**

Computer Science Student | DevOps Learner

## License

This project is intended for educational and learning purposes.
