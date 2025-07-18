# Red Hat Ansible Automation Platform 2.5 - Custom Execution Environment

This package contains all the necessary files to build a custom execution environment for Red Hat Ansible Automation Platform 2.5 using Ansible Builder.

## Directory Structure

```
custom-ee/
├── execution-environment.yml
├── requirements.yml
├── requirements.txt
├── bindep.txt
├── files/
│   ├── ansible.cfg
│   └── ca-bundle.crt
└── README.md
```

## Files Content

### execution-environment.yml

```yaml
---
version: 3

build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '--pre'

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest

dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt

additional_build_files:
  - src: files/ansible.cfg
    dest: configs
  - src: files/ca-bundle.crt
    dest: configs

additional_build_steps:
  prepend_base:
    # Copy CA certificate and update trust store
    - COPY _build/configs/ca-bundle.crt /etc/pki/ca-trust/source/anchors/
    - RUN update-ca-trust

  prepend_galaxy:
    # Configure Ansible Galaxy to use private automation hub
    - COPY _build/configs/ansible.cfg ~/.ansible.cfg
    # Set Galaxy environment variables
    - ENV ANSIBLE_GALAXY_SERVER_LIST=automation_hub
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_URL=https://ans-01.corp.ritcsusa.com/api/galaxy/
    - ENV ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_AUTH_URL=https://ans-01.corp.ritcsusa.com/api/galaxy/
    # Define build argument for authentication token
    - ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN

options:
  package_manager_path: /usr/bin/microdnf
```

### requirements.yml

```yaml
---
collections:
  - name: redhat.rhel_system_roles
    version: ">=1.0.0"
```

### requirements.txt

```
# Python dependencies for the execution environment
psutil>=5.8.0
six>=1.16.0
requests>=2.25.0
```

### bindep.txt

```
# System package dependencies
libxml2-devel [platform:rpm]
libxslt-devel [platform:rpm]
openssl-devel [platform:rpm]
```

### files/ansible.cfg

```ini
[galaxy]
server_list = automation_hub

[galaxy_server.automation_hub]
url = https://ans-01.corp.ritcsusa.com/api/galaxy/
auth_url = https://ans-01.corp.ritcsusa.com/api/galaxy/
token = ${ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN}
```

### files/pip.conf

*Note: This file is only needed for disconnected/offline environments. For standard connected builds, pip will use the default PyPI repositories and this file can be omitted.*

```ini
[global]
index-url = https://pypi.corp.ritcsusa.com/simple/
trusted-host = pypi.corp.ritcsusa.com
timeout = 60
```

### files/ca-bundle.crt

```
-----BEGIN CERTIFICATE-----
# Replace this with your actual CA certificate content
# This is a placeholder for your organization's CA certificate
# The certificate should be in PEM format
MIIEXjCCA0agAwIBAgIJAK8+YourCAContent+Here
... (certificate content) ...
-----END CERTIFICATE-----
```

## Build Instructions

### Prerequisites

1. Install Ansible Builder:
   ```bash
   sudo dnf install ansible-builder
   ```

2. Ensure you have access to:
   - Red Hat Container Registry (registry.redhat.io)
   - Your private automation hub (ans-01.corp.ritcsusa.com)
   - Internet access for Python package downloads (PyPI)

3. Set up authentication token for automation hub:
   ```bash
   export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN="your-token-here"
   ```

### Building the Execution Environment

1. Navigate to the project directory:
   ```bash
   cd custom-ee
   ```

2. Build the execution environment:
   ```bash
   ansible-builder build \
     --tag ans-01.corp.ritcsusa.com/custom-ee:latest \
     --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN="${ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN}" \
     --verbosity 2
   ```

3. Verify the image was created:
   ```bash
   podman images | grep custom-ee
   ```

### Pushing to Private Automation Hub

1. Log in to your private automation hub:
   ```bash
   podman login ans-01.corp.ritcsusa.com
   ```

2. Push the image:
   ```bash
   podman push ans-01.corp.ritcsusa.com/custom-ee:latest
   ```

### Testing the Execution Environment

Test that the execution environment works correctly:

```bash
podman run --rm -it ans-01.corp.ritcsusa.com/custom-ee:latest ansible-doc -l redhat.rhel_system_roles
```

## Configuration Notes

### CA Certificate
- Replace the placeholder content in `files/ca-bundle.crt` with your actual organizational CA certificate
- The certificate must be in PEM format
- Multiple certificates can be concatenated in this file if needed

### Authentication
- The `ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN` should be obtained from your automation hub
- This token provides access to certified collections

### Internal PyPI Mirror (Optional - Disconnected Environments Only)
- The `pip.conf` file is only needed for disconnected/offline environments
- For standard connected builds, Python packages will be downloaded from PyPI automatically
- If you need offline support, uncomment the pip.conf configuration in the execution-environment.yml

### System Dependencies
- Modify `bindep.txt` to include any additional system packages required by your collections
- Use the format `package-name [platform:rpm]` for RHEL/Rocky Linux packages

## Troubleshooting

### Common Issues

1. **SSL Certificate Errors**: Ensure your CA certificate is properly formatted and installed
2. **Authentication Failures**: Verify your automation hub token is valid and has proper permissions
3. **Network Connectivity**: Ensure access to all required repositories and registries
4. **Package Dependencies**: Check that all required system packages are available in your repositories

### Debugging

To debug build issues, use increased verbosity:

```bash
ansible-builder build --tag custom-ee:debug --verbosity 3
```

To inspect the generated Containerfile:

```bash
ansible-builder create --file execution-environment.yml
cat context/Containerfile
```

## Usage in Ansible Automation Platform

After successfully building and pushing your execution environment:

1. Log in to your Ansible Automation Platform web interface
2. Navigate to **Administration** → **Execution Environments**
3. Click **Add** to create a new execution environment
4. Set the **Image** field to: `ans-01.corp.ritcsusa.com/custom-ee:latest`
5. Configure any additional settings as needed
6. Save the execution environment

The custom execution environment can now be used in job templates and workflows.
