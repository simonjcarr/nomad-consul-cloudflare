# Infrastructure Ansible Playbook: Traefik + Nomad + Consul

## Pre-Deployment Configuration

Before running this playbook, you **must** review and update several configuration files to match your environment. These are primarily located in the `inventory/` and `group_vars/all/` directories.

**1. Inventory (`inventory/hosts.ini`):**
   - **Host IP Addresses:** Update the `ansible_host` for each server (e.g., `nomad-srv-1`, `nomad-client-1`, etc.) with the correct IP addresses for your target machines.
     ```ini
     [servers]
     nomad-srv-1 ansible_host=YOUR_SERVER_1_IP
     # ... and so on for other servers and clients
     ```
   - **SSH User:** Change `ansible_user` to the username you will use to connect to the servers via SSH.
     ```ini
     [all:vars]
     ansible_user = your_ssh_username
     ```
   - **SSH Private Key:** Modify `ansible_ssh_private_key_file` to point to the path of your SSH private key used for accessing the servers.
     ```ini
     ansible_ssh_private_key_file = /path/to/your/ssh_private_key
     ```

**2. Global Variables (`group_vars/all/main.yml`):**
   - **Datacenter Name:** `datacenter` (default: `"home_lab"`) can be customized to reflect your environment.
   - **Cloudflare Tunnel Domain:** `cloudflare_tunnel_domain` (e.g., `"*.example.com"`) **must** be changed to your own domain managed in Cloudflare. This is critical for the Cloudflare Tunnel functionality.
   - **Nomad Cloudflare Tunnel Job Directory:** `nomad_cloudflare_tunnel_job_dir` (default: `"/home/{{ ansible_user }}/jobs"`) defines where Nomad job files for the tunnel will be stored. Adjust if you have a different preferred path.

**3. Cloudflare Credentials (`group_vars/all/cloudflare.yml`):**
   - **Important:** You will likely need to copy `group_vars/all/cloudflare-example.yml` to `group_vars/all/cloudflare.yml` (or ensure your Cloudflare variables are defined in a file Ansible loads from `group_vars/all/`).
   - **`tunnel_token`**: This **must** be populated with your Cloudflare Tunnel token.
   - **`tunnel_id`**: This **must** be populated with your Cloudflare Tunnel ID.

**4. Traefik Configuration (`group_vars/all/traefik.yml`):**
   - **Traefik Config Directory:** `traefik_config_dir` (e.g., `"/home/{{ ansible_user }}/traefik-config"`) specifies where Traefik's static configuration files will be stored on the host. Update this to your preferred path.
   - **Traefik Nomad Job Path:** `traefik_nomad_job_path` (e.g., `"/home/{{ ansible_user }}/jobs/traefik.nomad"`) specifies where the generated Traefik Nomad job file will be stored. Update this to your preferred path.

**5. Cloudflare Tunnel Ingress Rules (Optional Customization):**
   - The file `roles/nomad_cloudflare_tunnel/templates/cloudflared_config.yml.j2` defines ingress rules.
   - Specifically, the hostname `nomad-test.carrtech.dev` is configured. You may want to change this subdomain or remove/add other specific hostnames based on your needs. The general `{{ cloudflare_tunnel_domain }}` will cover wildcard access.

**Note on Ansible Vault:** If you plan to store sensitive information like the `tunnel_token` securely, consider using Ansible Vault to encrypt `group_vars/all/cloudflare.yml` or a dedicated secrets file.

Once these configurations are tailored to your setup, you can proceed with running the playbook.

## VM Prerequisites

Before running the Ansible playbook, ensure your target Virtual Machines (VMs) meet the following prerequisites:

**1. Operating System:**
   - This playbook is primarily tested on Debian 12 (Bookworm) and was originally written for Ubuntu 24.04. Other Debian/Ubuntu-based distributions might work with minor adjustments.

**2. User Account & Sudo:**
   - A user account must exist on each VM that matches the `ansible_user` variable defined in your `inventory/hosts.ini` file (e.g., `simon`).
   - This user **must** have passwordless `sudo` privileges. To configure this:
     1. Log into the VM and run `sudo visudo`.
     2. Add the following line at the end of the file, replacing `your_username` with the actual `ansible_user`:
        ```
        your_username ALL=(ALL) NOPASSWD:ALL
        ```
     3. Save and exit the editor. This change allows Ansible to perform administrative tasks without prompting for a password.

**3. SSH Access:**
   - The `ansible_user` must be able to SSH into each VM using key-based authentication, with the public key present in the user's `~/.ssh/authorized_keys` file on the target VMs, and the corresponding private key specified by `ansible_ssh_private_key_file` in `inventory/hosts.ini`.

**4. Required Directory Structure:**
   - **`/mnt/data`**: This directory is crucial, especially on Nomad client nodes. It's used for persistent data for services like Traefik (e.g., configuration, ACME certificates). It is highly recommended to have this as a separate mount point, ideally managed by LVM (Logical Volume Management) for future expandability. The playbook assumes this path exists and is writable by services managed by Nomad.
   - **`/home/{{ ansible_user }}/jobs/`**: (Assuming `nomad_cloudflare_tunnel_job_dir` and `traefik_nomad_job_path` point here). This directory on the Ansible control node (or wherever the job files are templated to) will store generated Nomad job files.
   - **`/home/{{ ansible_user }}/traefik-config/`**: (Assuming `traefik_config_dir` points here). This directory on the Ansible control node (or relevant host if deploying Traefik's config directly) will store Traefik's static configuration files.

**5. Network Configuration:**
   - Ensure your VMs have static IP addresses assigned, corresponding to what you will define in the `inventory/hosts.ini` file.
   - VMs should have internet access for downloading packages and container images.
   - Firewall: Ensure that necessary ports are open between the nodes (e.g., for Consul, Nomad, and your applications) and from your load balancer to the Nomad clients. The playbook attempts to manage `ufw`, but your underlying network/cloud provider firewall might also need configuration.

**6. System Time (NTP):**
   - It's highly recommended to have NTP configured and synchronized on all VMs. Time mismatches can cause issues with distributed systems like Consul and Nomad (e.g., certificate validation, log correlation).

The playbook will attempt to install necessary software (like Docker, Nomad, Consul), create other required directories (like `/var/lib/nomad`, `/opt/consul`, `/etc/nomad.d`, etc.), and configure services. However, the prerequisites above are generally expected to be in place before the playbook runs.

## Overview
This Ansible playbook automates the deployment and configuration of a modern load balancing stack using Traefik, Nomad, and Consul. It is designed for a multi-node environment, with security and maintainability as priorities.

### Components
- **Traefik**: Edge router/load balancer, deployed as a Docker container via Nomad.
- **Nomad**: HashiCorp's workload orchestrator, manages container scheduling and lifecycle.
- **Consul**: Service discovery and health checking, integrated with Traefik.

## Playbook Structure
- **site.yml**: Main entry point, applies roles to the appropriate hosts.
- **roles/traefik**: Installs and configures Traefik, including Nomad job spec and config templates.
- **roles/clients/nomad**: Handles Nomad client configuration, including Docker integration.
- **roles/common**: Shared tasks for all nodes.

## Special Attention: Docker Volumes with Nomad

### The Problem
By default, Nomad does **not** allow Docker jobs to mount arbitrary host paths for security reasons. If you attempt to mount a host path (e.g. `/mnt/data/traefik-config/traefik.yml:/etc/traefik/traefik.yml`) in your Nomad job spec, you will get an error like:

```
volumes are not enabled; cannot mount host paths
```

### The Solution
You **must** enable host path volumes in the Nomad Docker driver config on every Nomad client. This is done by adding the following block to `/etc/nomad.d/client.hcl`:

```hcl
plugin "docker" {
  config {
    volumes {
      enabled = true
    }
  }
}
```
After updating this file, restart the Nomad client service:

```
sudo systemctl restart nomad
```

### Why This Matters
If this is not set, any Nomad job using direct Docker host path mounts will fail to start, even if it works with `docker run` manually.

## Traefik Job Spec Example
Your `traefik.nomad.j2` should use **absolute host paths** for volumes, for example:

```hcl
config {
  image = "traefik:v2.11"
  ports = ["web"]
  volumes = [
    "/mnt/data/traefik-config/traefik.yml:/etc/traefik/traefik.yml",
    "/mnt/data/traefik-config/acme.json:/acme.json"
  ]
  network_mode = "host"
}
```

## Troubleshooting
- If Traefik works with `docker run` but fails in Nomad, check the Docker volume configuration above.
- Resource settings (CPU/memory) can be kept low; the main blocker is usually the Docker volume permission.
- Allocation logs are your friend: check them in the Nomad UI or with `nomad alloc logs <alloc_id>`.

## Summary of Key Steps
1. Ensure `/mnt/data/traefik-config` exists and is accessible on all Nomad clients.
2. Enable Docker host path volumes in Nomad client config as shown above.
3. Use absolute paths for volumes in the Nomad job spec.
4. Deploy with Ansible (`ansible-playbook -i inventory/hosts.ini site.yml`).
5. Monitor Nomad UI for allocation status and logs.

## Contact
For further issues or contributions, please open an issue or PR in this repository.
