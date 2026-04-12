# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Role Does

This Ansible role ingests directories of `compose.yaml` files from a controller host and outputs a single merged, sanitised `compose.yaml` to a remote host. It supports Jinja2/Ansible variable interpolation within compose files, selective disabling of services, config file deployment via `config-*` directories, and optional ZFS dataset creation.

## Commands

### Run all tests
```bash
cd tests && pip install ansible pytest && python -m pytest test_yaml_indent.py -v && ansible-playbook test-playbook.yml -v
```

### Run only filter plugin unit tests
```bash
cd tests && python -m pytest test_yaml_indent.py -v
```

### Run only the Ansible integration test
```bash
cd tests && ansible-playbook test-playbook.yml -v
```

### Run a single pytest test
```bash
cd tests && python -m pytest test_yaml_indent.py::TestIndentYamlLists::test_simple_list_indentation -v
```

## Architecture

### Core flow (`tasks/main.yml`)
1. Finds subdirectories under `services/<hostname>/` (each is a named stack), excluding `config-*/` directories
2. For each stack, calls `generate-stack.yml` with the stack subdirectory as the search path and `docker_compose_generator_output_path/<stack>/` as the output path
3. Within each stack: uses `filetree` to find all `.yaml`/`.yml` files, renders them through the template engine (enabling variable interpolation), merges all `services`, `networks`, `volumes`, `configs`, and `secrets` sections using `combine()`
4. Applies the custom `indent_yaml_lists` filter to fix list indentation (Ansible's `to_nice_yaml` outputs lists unindented under their keys)
5. Writes the merged result to `docker_compose_generator_output_path/<stack>/compose.yaml`
6. Separately finds `config-*` directories and delegates config deployment to `tasks/deploy-config.yml`
7. Copies any `.env`/`env` file found in the stack source directory to the stack output directory

### Custom filter plugin (`filter_plugins/yaml_indent.py`)
The `indent_yaml_lists` filter post-processes YAML strings to indent list items 2 spaces under their parent key. This is necessary because `to_nice_yaml` produces non-standard indentation for lists. The filter tracks state as it scans lines, indenting list items when they appear at the same level as the preceding key.

### Config deployment (`tasks/deploy-config.yml`)
For each `config-*` directory found: reads a `.dest` file (supports Ansible variable interpolation) for the target path, optionally creates a ZFS child dataset, creates the destination directory, then templates all non-`.dest` files to the destination.

### Key variables (`defaults/main.yml`)
| Variable | Default | Purpose |
|---|---|---|
| `docker_compose_generator_output_path` | `~/docker` (invoking user's home when using become) | Where to write stack subdirs on the remote host |
| `docker_compose_generator_uid` | Connecting user's UID; `SUDO_UID` when using become | Owner UID for written files |
| `docker_compose_generator_gid` | Connecting user's primary GID; `SUDO_GID` when using become | Owner GID for written files |
| `services_directory` | `{{ playbook_dir }}/services/` | Root of services tree on the controller |
| `docker_compose_hostname` | (unset) | Override for directory name matching; defaults to `inventory_hostname` |
| `disabled_compose_files` | (unset) | List of service directory names to exclude |
| `docker_compose_generator_zfs_enabled` | `false` | Enable ZFS dataset creation for config dirs |

### Test structure (`tests/`)
- `test_yaml_indent.py` — pytest unit tests for the filter plugin, imported directly from `filter_plugins/`
- `test-playbook.yml` — full Ansible integration test that runs the role against `services/test-app/` and asserts on the generated output
- `services/test-stack-app/` — test fixture; contains `web/` and `data/` subdirectories each with their own `compose.yaml` and `config-*` directories
