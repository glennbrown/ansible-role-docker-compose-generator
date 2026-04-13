# ansible-role-docker-compose-generator

This role ingests directories of `compose.yaml` files from a controller host and outputs a merged, sanitised `compose.yaml` to each remote host using Ansible. It supports Jinja2/Ansible variable interpolation within compose files, selective disabling of services, config file deployment, and optional ZFS dataset creation.

## Usage

Import this role into your Ansible setup either as a git submodule or via Ansible Galaxy.

In the root of your git repo create a `services` directory structured as follows:

```
services/
└── ansible-hostname1
    ├── web
    │   ├── nginx
    │   │   └── compose.yaml
    │   └── certbot
    │       └── compose.yaml
    └── data
        ├── postgres
        │   └── compose.yaml
        └── redis
            └── compose.yaml
```

Each subdirectory directly under `services/<hostname>/` is a **stack**. All `compose.yaml` files within a stack are merged together and written to a matching subdirectory of the output path. Services are never merged across stacks.

Each `compose.yaml` is a standard Docker Compose file. Ansible variable interpolation is supported:

```yaml
services:
  librespeed:
    image: lscr.io/linuxserver/librespeed
    container_name: librespeed
    ports:
      - 8008:80
    environment:
      - "TZ={{ host_timezone }}"
      - "PASSWORD={{ testpass }}"
    restart: unless-stopped
```

Variables can come from Ansible Vault, `host_vars`, `group_vars`, or any other standard Ansible variable source. Multiple services per compose file are supported.

By default every `compose.yaml` found under a stack is included. To exclude specific services, set the following in `host_vars` or `group_vars`, matching the **directory name** of the service within the stack:

```yaml
disabled_compose_files:
  - jellyfin
```

## Output

Each stack produces its own `compose.yaml` under a matching subdirectory of `docker_compose_generator_output_path`:

```
docker_compose_generator_output_path/   # defaults to ~/docker
├── web
│   └── compose.yaml
└── data
    └── compose.yaml
```

The output path defaults to `~/docker` in the home directory of the connecting user. When using `become: true` (sudo), it defaults to the home directory of the user who invoked sudo, not root. Override with:

```yaml
docker_compose_generator_output_path: /opt/docker
```

### `.env` files

If an `.env` or `env` file exists in a stack source directory, it is templated and copied to the stack's output directory as `.env`. This is useful for passing secrets or environment-specific values to Docker Compose at runtime.

### Secret files

Files named `secret-<name>.<ext>` in a stack source directory are templated and deployed alongside `compose.yaml`, with the `secret-` prefix stripped:

| Source | Deployed as |
|---|---|
| `secret-cloudflare.txt` | `cloudflare.txt` |
| `secret-db_password.env` | `db_password.env` |

Ansible variable interpolation is supported inside secret files. Reference them in your compose file using a relative path:

```yaml
secrets:
  cloudflare_token:
    file: ./cloudflare.txt
```

## Custom hostnames

By default the role looks for a directory under `services/` matching your Ansible inventory hostname. Override this with:

```yaml
docker_compose_hostname: my-custom-hostname
```

## Override services directory location

By default `services_directory` is `{{ playbook_dir }}/services/`. Override it if your playbooks live in a subdirectory:

```yaml
services_directory: /path/to/my/services/
```

## Config File Deployment

Place a `config-<appname>` directory alongside your compose files to deploy configuration files to the remote host:

```
services/
└── ansible-hostname1
    └── monitoring
        ├── compose.yaml
        ├── config-prometheus
        │   ├── prometheus.yml
        │   └── rules
        │       └── alerts.yml
        └── config-grafana
            ├── .dest
            └── grafana.ini
```

### Destination path

Add a `.dest` file to the `config-<appname>` directory to specify an explicit destination path on the remote host. Ansible variable interpolation is supported:

```
{{ appdata_path }}/apps/grafana
```

If `.dest` is omitted the role computes a default destination automatically:

- **One** `config-*` directory in the stack → `<stack_output_path>/config/`
- **Multiple** `config-*` directories in the stack → `<stack_output_path>/config/<appname>/`

### File handling

All files inside `config-<appname>/` (except `.dest`) are deployed to the destination. Subdirectory structure is preserved. Files are processed through Ansible's template engine, so Ansible variable interpolation works inside config files.

### Optional: ZFS Dataset Creation

If your target host uses ZFS, you can enable automatic dataset creation for config directories:

```yaml
# group_vars or host_vars
docker_compose_generator_zfs_enabled: true
```

When enabled, the role will:
1. Detect the parent ZFS dataset of the destination path
2. Create a child dataset for the config directory
3. The directory is then auto-mounted by ZFS

This provides snapshot and replication benefits for your app configs.
