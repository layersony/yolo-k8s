# YOLO - E-commerce Application

A full-stack e-commerce application built with Node.js, Express, MongoDB, and React. 
It Demonstrates modern web development practices with containerized deployment using Docker, automated provisioning with Vagrant and configuration management with Ansible

## Features
- Product Management - List, Add, Edit and Delete Products
- Modern React Frontend
- Restful API Backend
- Persistent Data Storage with MongoDB
- Automated Infrasturture Provisioning
- Configuration management with Ansible

## Tech Stack

### Frontend
- React.js - UI library
- CSS3 - Styling
- HTML5 - Markup

### Backend
- Node.js - Runtime environment
- Express.js - Web framework
- MongoDB - NoSQL database
- Mongoose - MongoDB object modeling

### DevOps
- Docker - Containerization
- Docker Compose - Multi-container orchestration
- Vagrant - Development environment and automation
- Ansible - Configuration management and automation
- Virtualbox - Virtualization Provider


## Project Structure
```
yolo/
|-- vagrant/
│   -- Vagrantfile  # VM configuration and provisioning
│   -- .vagrant/  # Vagrant metadata (auto-generated)
|
|-- ansible/
│   |-- ansible.cfg  # Ansible configuration
│   |-- group_vars/
│   |   -- all.yml   # Global variables for all environments
│   |-- inventory/
│   │   -- development/
│   │       -- hosts.ini    # Development environment hosts
│   |-- playbooks/
│   |     -- deploy-dev.yml   # Main deployment playbook
│   |-- roles/    # Ansible roles for each component
│        -- backend/
│        -- common/
│        -- docker/
│        -- frontend/
│        -- mongodb/
|
|-- backend/
|-- client/
|-- .gitignore
|-- backend.png
|-- frontend.png
|-- docker-compose.yml
|-- LICENSE
|-- README.md
```

## Prerequisites

- Node.js v18+
- npm 
- MongoDB v4.4+
- Docker and Docker Compose
- Vagrant v2.0+
- Virtualbox v6.0+
- Ansible v2.9+

## Installation & Deployment Options

Clone Repo
```bash

git clone https:github.com/layersony/yolo.git

```

## For Setups

###  Option 1

Will Be Using Vagrant + Ansible

*TO NOTE*

The VM will use this resources:
- RAM: 1024 MB
- CPUs: 2 cores
- Disk: Dynamic allocation

#### Step By Step Procedure

```bash

cd yolo

# navigate to vagrant directory
cd vagrant

# Start and Provisioning VM
vagrant up

# the above command will
# Create an Ubuntu 22.04 on VM
# Configure networking and port forwarding
# Install Docker and all dependencies
# Deploy MongoDB, Backend, and Frontend containers
# Configure the entire application stack

```

The access points once provisioned will be:

- Frontend: http://localhost:3000
- Backend API: http://localhost:5001
- MongoDB: mongodb://localhost:27017
- VM IP: `192.168.56.10`
- NAT (for Internet Access)

### Ports Forwarding

| Service | VM Port | Host Port|
|---------|---------|----------|
| Frontend | 3000 | 3000|
| Backend | 5001 | 5001 |
| MongoDB | 27017 | 27017|


### Option 2


   
## Docker Deployment
For quick deployment without VM overhead:

### Quick Start with Docker

1. **Build and start all services**
   ```bash
   docker-compose up --build
   ```

2. **Access the application**
   - Frontend: `http://localhost:80`
   - Backend API: `http://localhost:5001`
   - MongoDB: `mongodb://localhost:27017`

The application consists of three services:

- mongodb: MongoDB database server (port 27017)
- backend: Express.js API server (port 5001)
- frontend: React development server (port 80)

### Docker Commands

```bash
# Start services in detached mode
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f

# Rebuild specific service
docker-compose up --build backend

# Remove all containers and volumes
docker-compose down -v
```

## Testing
For Vagrant + Ansible

```bash
# SSH into VM first
vagrant ssh

# Run backend tests
cd /opt/yolo-app/backend
npm test

# Run frontend tests
cd /opt/yolo-app/client
npm test

# Exit VM
exit
```

For Docker

```bash
# Run backend tests
cd backend
npm test

# Run frontend tests
cd client
npm test
```

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details

## Authors

- layersony - [GitHub Profile](https://github.com/layersony)

## Acknowledgments

- Original project forked from [kadimasum/yolo](https://github.com/kadimasum/yolo)
- Built with the MERN stack
- Inspired by modern e-commerce platforms