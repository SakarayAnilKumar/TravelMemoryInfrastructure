# MERN Stack Deployment on AWS Using Terraform and Ansible: Implementation Report

## 1. Executive Summary & Assignment Overview
* This implementation document outlines the complete architecture, automation workflows, and configuration details for deploying the **TravelMemory MERN Stack Application** on Amazon Web Services (AWS).
* The project fulfills all operational criteria by leveraging **Terraform** for reproducible infrastructure provisioning and **Ansible** for automated configuration management and application deployment.
* The complete stack integrates an Nginx-backed React frontend, a Node.js/Express backend daemonized via PM2, and an isolated MongoDB database running on a private subnet.

---

## 2. Infrastructure Architecture & Network Topology
* **Amazon VPC Design**: The infrastructure is provisioned inside a custom VPC configured with both public and private subnets to isolate database operations from public exposure.
* **Public Subnet (Web Server)**: Hosts an EC2 instance running Nginx and the Node.js/Express backend, connected directly to an Internet Gateway to accept inbound HTTP/HTTPS and SSH traffic.
* **Private Subnet (Database Server)**: Hosts an EC2 instance running MongoDB, completely blocked from direct external access and reachable only via internal VPC routing or an SSH proxy jump host.
* **Routing & Security Groups**: Network ACLs and security groups restrict inbound database traffic (port 27017) and SSH traffic (port 22) exclusively to internal web server IP ranges.

---

## 3. Prerequisites & Control Node Environment
* **Local Control Machine**: Operates as the central runner executing Terraform templates and Ansible playbooks against target AWS endpoints.
* **Authentication & Credentials**: AWS CLI authenticated with explicit IAM permissions, utilizing a shared private key pair (`anil-us-east-keypair.pem`) for secure instance access.
* **Tooling Versions**: Requires Terraform (v1.5+) and Ansible (v2.10+) installed locally with required community collections (`ansible.posix`, `ansible.builtin`).

---

## 4. Part 1: Infrastructure as Code (IaC) Approach & File Breakdown
The infrastructure provisioning follows modular Terraform principles. Terraform manages cloud resources declaratively, establishing dependency chains (e.g., subnets depend on VPCs, route tables depend on gateways, and EC2 instances depend on subnets and security groups).

Below is the detailed breakdown of the infrastructure components and files typically found in your repository:

### Core Terraform Files Explained
1. **`providers.tf` (or `main.tf`)**
   * *Purpose*: Configures the required cloud provider (AWS), specifies the target region (`us-east-1`), and sets minimum version requirements for Terraform and provider plugins.
2. **`vpc.tf` & `subnets.tf`**
   * *Purpose*: Creates a custom Virtual Private Cloud (VPC) with a defined CIDR block. It provisions **two distinct subnets** as required by the assignment:
     * **Public Subnet**: Configured with an Internet Gateway (`igw`) attachment and public IP auto-assignment, housing the web server instance.
     * **Private Subnet**: Configured with a NAT Gateway (or route table adjustments) so internal resources can route traffic securely without direct public exposure, housing the database server.
3. **`security_groups.tf`**
   * *Purpose*: Acts as virtual firewalls for your instances.
     * **Web Security Group**: Allows inbound HTTP traffic on port `80`, SSH traffic on port `22` (restricted to administrative IPs), and outbound internet access.
     * **Database Security Group**: Restricts inbound traffic on port `27017` (MongoDB) and port `22` (SSH) *strictly* to traffic originating from the Web Server's private IP or security group ID, completely blocking public access.
4. **`ec2.tf`**
   * *Purpose*: Provisions the actual compute instances.
     * **Web Server Instance**: Placed in the public subnet using an AMI (like Amazon Linux or Ubuntu) and attached to the web security group.
     * **Database Server Instance**: Placed in the private subnet and attached to the database security group. It references your shared key pair (`anil-us-east-keypair.pem`) for administrative access.
5. **`outputs.tf`**
   * *Purpose*: Exposes critical attributes after `terraform apply` completes—specifically outputting the **public IP address of the web server instance** so you can easily reference it in your Ansible inventory.

### Essential Terraform Commands Used
```bash
# 1. Initialize the Terraform workspace and download required AWS provider plugins
terraform init

# 2. Check configuration syntax and validate files
terraform validate

# 3. Preview all AWS resources that will be created or modified
terraform plan

# 4. Provision the cloud infrastructure automatically without manual confirmation prompts
terraform apply --auto-approve

# 5. View output variables (such as the web server public IP address)
terraform output

# 6. Tear down all infrastructure resources when cleaning up the environment
terraform destroy --auto-approve
```

---

## 5. Part 2: Configuration Management & Automation Approach (Ansible Breakdown)
Once Terraform provisions the bare-metal EC2 instances, Ansible takes over configuration management, software installation, and code deployment. 

### Core Ansible Files & Structure Explained
1. **`inventory/hosts.ini` (Dynamic/Static Inventory)**
   * *Purpose*: Maps out target servers into functional groups (`[webservers]` and `[dbservers]`).
   * *ProxyCommand Implementation*: Because the database server resides in a private subnet with no public IP, the inventory configures an SSH `ProxyCommand` tunnel. This instructs Ansible to route database connection commands transparently through the public `web-server` acting as a jump host.
2. **`site.yml` (Master Playbook)**
   * *Purpose*: Acts as the orchestration entry point, calling out execution order across defined roles (e.g., executing web-server setup tasks and database configurations in a structured sequence).
3. **`roles/web-server/tasks/main.yaml` (Web Configuration Tasks)**
   * *System Dependencies*: Installs `git`, `nginx`, and sets up the NodeSource repository to install `Node.js` and `npm`.
   * *PM2 Installation*: Installs the `pm2` process manager globally via npm to keep the Node.js backend alive continuously.
   * *Repository Management*: Clones the `TravelMemory` source code repository from GitHub into `/home/ec2-user/TravelMemory` with correct file permissions (`ec2-user`).
   * *Backend Setup*: Installs npm packages inside the backend folder, auto-injects dotenv configurations, creates a dynamic `.env` file mapping `PORT=3001` and `MONGO_URI=mongodb://<private_db_ip>:27017/travelmemory`, and starts the process using PM2.
   * *Frontend Build & Nginx Routing*: Compiles the React application bundle (`npm run build`), copies static assets to Nginx's web root (`/var/www/travelmemory`), writes a custom Nginx virtual host configuration (`travelmemory.conf`) to serve static content on port `80` while proxying `/api/` requests directly to `http://localhost:3001/`, and restarts Nginx.

### Essential Ansible Commands Used

``` bash
# 1. Test basic network connectivity and SSH reachability for all inventory hosts
ansible all -i inventory.ini -m ping

# 2. Test connectivity specifically through the proxy jump host to the private database server
ansible dbservers -i inventory.ini -m ping

# 3. Run a syntax check on your master playbook before execution
ansible-playbook -i inventory.ini playbook.yaml --syntax-check

# 4. Perform a dry-run execution (simulate changes without applying them)
ansible-playbook -i inventory.ini playbook.yaml --check

# 5. Execute the master configuration and deployment playbook
ansible-playbook -i inventory.ini playbook.yaml
```
---

## 6. Application Integration, Runtime & Reverse Proxy Routing
The application execution relies on clean separation of duties and local network routing between components:

1. **Frontend Request Flow**: 
   * When a user visits the web server's public IP in a browser, **Nginx** handles the incoming HTTP request on port `80`.
   * Nginx serves the pre-compiled, static React production assets (`index.html`, JavaScript bundles, CSS) directly from `/var/www/travelmemory` with high performance.
2. **API & Backend Execution**:
   * User interactions in the React UI trigger API requests to `/api/...`.
   * **Nginx Reverse Proxy** intercepts any URL matching location `/api/` and transparently forwards the request internally to `http://localhost:3001/`.
   * The **Node.js/Express backend**, running continuously in the background managed by **PM2**, receives the request on port `3001`.
3. **Database Interaction**:
   * The Express application queries data using the Mongoose ODM. 
   * Utilizing the `MONGO_URI` environment variable established during Ansible deployment, the backend talks securely across the private VPC network to the **MongoDB database instance** running on the private subnet (`10.0.x.x:27017`).
   * Because MongoDB is locked inside the private subnet and protected by security groups, data transactions remain completely isolated from public exposure while remaining fully accessible to the backend application.

---

## 7. Security Hardening & Operational Best Practices
* **Network Isolation**: MongoDB instances reside entirely in private subnets, completely hidden from public enumeration or direct attack vectors.
* **Privilege Separation & Permissions**: Application files and web roots enforce least-privilege ownership assigned explicitly to the `nginx` system user to prevent `13: Permission denied` errors.
* **Credential Protection**: Sensitive connection strings and secrets are decoupled from source code, injected purely through runtime environment variables and protected SSH keys.

---

## 8. Verification, Testing & Deliverables
* **Service Health Checks**: Verified via `sudo systemctl status nginx` and `pm2 status` to confirm active daemon states across target servers.
* **End-to-End Functional Test**: Validated browser accessibility using the web server public IP address, confirming seamless data fetching from the Express/MongoDB backend tier.
* **Repository Deliverables**: All Terraform configuration scripts, Ansible playbooks, inventory files, and this documentation have been packaged and pushed to GitHub as required by assignment guidelines.