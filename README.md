# Infrastructure Ansible Playbook: Traefik + Nomad + Consul

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
