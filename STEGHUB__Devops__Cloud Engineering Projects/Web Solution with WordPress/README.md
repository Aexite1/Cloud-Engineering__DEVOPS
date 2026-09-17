# Web Solution with WordPress

This repository documents my hands-on implementation of a **WordPress web solution on AWS** using separate EC2 instances for the Web Server and Database Server.

The project covers server storage configuration with LVM, WordPress deployment, Apache and PHP setup, MySQL configuration, and the connection between the Web Server and Database Server.

## Repository Structure

```text
.
├── images/
├── resource.md
├── web_solution_with_wordpress.md
└── README.md
```

### 📁 images/

Contains screenshots captured throughout the step-by-step implementation of the project.

The screenshots are referenced in the main documentation to provide a visual guide to the different stages of the setup.

### 📄 web_solution_with_wordpress.md

This is the main project documentation.

It provides a detailed, step-by-step explanation of the implementation process, including the commands, configurations, LVM setup, server configuration, WordPress installation, MySQL setup, and Web Server-to-Database Server connection.

The screenshots in the `images/` folder are used throughout the documentation to support the explanations.

### 📄 resource.md

Contains useful code snippets and supporting resources used during the project.

## Project Overview

The project involved deploying WordPress with the application and database layers separated across two EC2 instances.

The **Web Server** hosts Apache, PHP, and the WordPress application, while the **Database Server** hosts MySQL. LVM was also used to organize storage on the servers.

### Technologies & Concepts

- AWS EC2
- Linux / Red Hat Enterprise Linux
- LVM
- Apache
- PHP
- WordPress
- MySQL
- Git & GitHub

## Recommended Reading Order

1. Start with `web_solution_with_wordpress.md`.
2. Follow the screenshots in `images/` alongside the documentation.
3. Refer to `resource.md` for useful snippets and supporting resources.

## Author

**Great Ojuolape**

Computer Science Student | DevOps/ Cloud Engineering Aspirant

## License

This project is intended for educational and learning purposes.
