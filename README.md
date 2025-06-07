# GitLab-CE with Podman and Systemd (Quadlets)

This guide details how to set up GitLab Community Edition (CE) using Podman, managed by systemd via Quadlet files. This setup includes a dedicated Podman network for GitLab and ensures GitLab starts automatically on system boot.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE.md) file for details.

## Prerequisites

1.  **Podman Installed:**
    ```bash
    # For Fedora/CentOS/RHEL
    sudo dnf install podman
    # For Debian/Ubuntu
    sudo apt update && sudo apt install podman
    ```
2.  **Sufficient System Resources:**
    *   **CPU:** At least 2 cores (4+ recommended).
    *   **RAM:** At least 4GB RAM free for the container (8GB+ total system RAM highly recommended).
    *   **Disk Space:** 20GB+ for GitLab itself, plus space for your repositories.
3.  **Hostname / DNS:**
    *   A hostname (e.g., `gitlab.yourdomain.com`) that resolves to your server's IP address.
    *   Replace `<your_gitlab_hostname>` in the configuration files with your actual hostname or IP.
4.  **Firewall:** You'll need to open ports on your host (see [Firewall Configuration](#firewall-configuration)).

## Setup Instructions

### 1. Prepare Host Directories for GitLab Data

These directories will store GitLab's configuration, logs, and application data, persisting it outside the container.

```bash
sudo mkdir -p /srv/gitlab/config
sudo mkdir -p /srv/gitlab/logs
sudo mkdir -p /srv/gitlab/data
# Set permissive permissions initially; can be tightened later after GitLab writes files.
sudo chmod -R 777 /srv/gitlab
```

### 2. Create Podman Network Quadlet File

This defines a dedicated bridge network for GitLab.

**File:** `/etc/containers/systemd/gitlab-net.network`
```ini
[Unit]
Description=Podman network for GitLab
After=network-online.target
Wants=network-online.target

[Network]
NetworkName=gitlab-net
Driver=bridge
# Optional: Specify subnet and gateway if needed
# Subnet=10.89.0.0/16
# Gateway=10.89.0.1

[Install]
WantedBy=multi-user.target
```
*Note: The `systemd-quadlet-generator` will likely create a service file named `gitlab-net-network.service` from this Quadlet file.*

### 3. Create GitLab Container Quadlet File

This file defines the GitLab container itself.

**File:** `/etc/containers/systemd/gitlab.container`
```ini
[Unit]
Description=GitLab CE Container
# Ensure the network service (likely gitlab-net-network.service) is started first
After=network-online.target gitlab-net-network.service
Wants=network-online.target gitlab-net-network.service

[Container]
# Use a fully qualified image name (docker.io is the default registry for gitlab/gitlab-ce)
Image=docker.io/gitlab/gitlab-ce:latest
ContainerName=gitlab

# --- IMPORTANT: CUSTOMIZE THE FOLLOWING ---
# PodmanArgs is used to pass arguments like --hostname to the underlying podman run command
PodmanArgs=--hostname=<your_gitlab_hostname>

# Environment variables for GitLab configuration
# - Replace <your_gitlab_hostname> in external_url
# - Replace <your_ssh_host_port> with the host port you map to container's port 22
Environment=GITLAB_OMNIBUS_CONFIG="external_url 'http://<your_gitlab_hostname>'; gitlab_rails['lfs_enabled'] = true; gitlab_rails['gitlab_shell_ssh_port'] = <your_ssh_host_port>;"

# Port mapping (host:container)
# Replace <http_host_port>, <https_host_port>, <ssh_host_port> with your desired host ports
PublishPort=<http_host_port>:80
PublishPort=<https_host_port>:443
PublishPort=<ssh_host_port>:22
# --- END OF CUSTOMIZATION SECTION ---

# Volume mounts (host_path:container_path:options)
# The :Z option handles SELinux labeling if SELinux is enabled.
Volume=/srv/gitlab/config:/etc/gitlab:Z
Volume=/srv/gitlab/logs:/var/log/gitlab:Z
Volume=/srv/gitlab/data:/var/opt/gitlab:Z

# Shared memory size recommended for GitLab
ShmSize=256m

# Connect to the dedicated network
Network=gitlab-net

[Service]
# How long systemd waits for the service to start
TimeoutStartSec=300
# Restart policy
Restart=always

[Install]
WantedBy=multi-user.target
```

**Example Customization for `gitlab.container`:**
If your GitLab hostname is `gitlab.example.com` and you want to use host ports 1080 (HTTP), 1443 (HTTPS), and 1222 (SSH), you would set:
*   `PodmanArgs=--hostname=gitlab.example.com`
*   `Environment=GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.example.com'; gitlab_rails['lfs_enabled'] = true; gitlab_rails['gitlab_shell_ssh_port'] = 1222;"`
*   `PublishPort=1080:80`
*   `PublishPort=1443:443`
*   `PublishPort=1222:22`

### 4. Reload Systemd and Start Services

```bash
# Reload systemd to recognize new/changed Quadlet files and generate services
sudo systemctl daemon-reload

# Enable and start the network service first
# (The generated service name might be gitlab-net-network.service)
sudo systemctl enable --now gitlab-net-network.service
sudo systemctl status gitlab-net-network.service # Verify it's active

# Enable and start the GitLab container service
sudo systemctl enable --now gitlab.service
sudo systemctl status gitlab.service # Verify it's active
```

## Post-Installation

### 1. Firewall Configuration

Open the host ports you mapped in `gitlab.container` (e.g., 1080, 1443, 1222).

**For `firewalld` (e.g., Fedora, CentOS, RHEL):**
```bash
sudo firewall-cmd --permanent --add-port=<http_host_port>/tcp
sudo firewall-cmd --permanent --add-port=<https_host_port>/tcp
sudo firewall-cmd --permanent --add-port=<ssh_host_port>/tcp
sudo firewall-cmd --reload
```

**For `ufw` (e.g., Debian, Ubuntu):**
```bash
sudo ufw allow <http_host_port>/tcp
sudo ufw allow <https_host_port>/tcp
sudo ufw allow <ssh_host_port>/tcp
sudo ufw reload # Or sudo ufw enable if not already enabled
```

### 2. GitLab First Run and Initial Password

*   GitLab will take **5-15 minutes (or more)** to initialize the first time.
*   Monitor its progress:
    ```bash
    sudo podman logs -f gitlab
    # Or via systemd
    sudo journalctl -fu gitlab.service
    ```
*   Once ready, find the initial `root` user password:
    ```bash
    sudo podman exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
    ```
    (This file is automatically deleted after 24 hours on the first run).
*   Access GitLab in your browser at `http://<your_gitlab_hostname>:<http_host_port>` (e.g., `http://gitlab.example.com:1080`).
*   Log in as `root` with the password retrieved. **Change this password immediately.**

## Managing GitLab Service

*   **Check status:**
    ```bash
    sudo systemctl status gitlab.service
    sudo systemctl status gitlab-net-network.service
    sudo podman ps -a | grep gitlab
    ```
*   **Start service:**
    ```bash
    sudo systemctl start gitlab.service
    ```
*   **Stop service:**
    ```bash
    sudo systemctl stop gitlab.service
    ```
*   **Restart service:**
    ```bash
    sudo systemctl restart gitlab.service
    ```
*   **View logs:**
    ```bash
    sudo journalctl -u gitlab.service
    sudo journalctl -fu gitlab.service # Follow logs
    sudo podman logs gitlab
    ```

## Important Considerations

*   **HTTPS/SSL:** For any production or publicly accessible instance, **you MUST configure HTTPS.**
    *   This typically involves setting `external_url 'https://<your_gitlab_hostname>:<https_host_port>'` in `gitlab.container` (or directly in `/srv/gitlab/config/gitlab.rb` followed by `sudo podman exec gitlab gitlab-ctl reconfigure`).
    *   You'll need to provide SSL certificates (e.g., via Let's Encrypt or your own). GitLab has built-in Let's Encrypt support, or you can use a reverse proxy.
*   **Email Configuration:** Configure GitLab to send emails for notifications, password resets, etc. Edit `/srv/gitlab/config/gitlab.rb` on the host and then run `sudo podman exec gitlab gitlab-ctl reconfigure`.
*   **Backups:** Regularly back up the `/srv/gitlab/` directory. GitLab also has built-in backup tools (`gitlab-backup create`).
*   **Updates:**
    1.  Pull the new image: `sudo podman pull docker.io/gitlab/gitlab-ce:latest` (or a specific version).
    2.  Stop GitLab: `sudo systemctl stop gitlab.service`.
    3.  Remove the old container (data is safe in volumes): `sudo podman rm gitlab`.
    4.  Start GitLab: `sudo systemctl start gitlab.service`. Podman (via systemd and the Quadlet's behavior) will recreate the container using the new image and existing configuration.
    5.  Monitor logs for database migrations or reconfigurations.

## Troubleshooting Common Issues

*   **Quadlet Generator Errors:** If `systemctl daemon-reload` has issues or service files are not generated in `/run/systemd/generator/`:
    *   Check for Quadlet syntax errors: `sudo journalctl -xe | grep -i quadlet`
    *   Common unsupported keys in `[Container]` section (use `PodmanArgs=` or move to `[Service]`):
        *   `Hostname`: Use `PodmanArgs=--hostname=<value>` in `[Container]`.
        *   `Restart`: Move `Restart=always` to the `[Service]` section.
    *   Ensure Quadlet files are in `/etc/containers/systemd/` and have correct extensions (`.container`, `.network`).
*   **Podman Exit Code 125 (Container Fails to Start):**
    *   Temporarily comment out `Restart=always` in `gitlab.container`'s `[Service]` section, run `sudo systemctl daemon-reload`, then `sudo systemctl start gitlab.service`.
    *   Check logs immediately: `sudo journalctl -u gitlab.service -n 100 --no-pager`. This usually shows the direct error from Podman.
    *   **"unable to find network with name or ID gitlab-net"**: Ensure `gitlab-net-network.service` is running (`sudo systemctl status gitlab-net-network.service`). If not, check its logs (`sudo journalctl -u gitlab-net-network.service`).
    *   **Port Conflicts**: Check if host ports are in use: `sudo ss -tulnp | grep -E ':(<http_port>|<https_port>|<ssh_port>)'`
    *   **Volume Path/Permission Issues**: Verify `/srv/gitlab/*` directories exist and are writable.
    *   **SELinux Denials**: `sudo ausearch -m avc -ts recent`. The `:Z` volume option should handle most cases.
*   **Verify Generated Service Names:**
    *   The network service from `gitlab-net.network` is likely `gitlab-net-network.service`.
    *   The container service from `gitlab.container` is likely `gitlab.service`.
    *   Check `/run/systemd/generator/` to confirm actual generated service names.

