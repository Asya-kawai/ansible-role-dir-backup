# Ansible Role For Backup web server settings

[![CI](https://github.com/Asya-kawai/ansible-role-backup/actions/workflows/ci.yml/badge.svg)](https://github.com/Asya-kawai/ansible-role-backup/actions/workflows?query=workflow%3ACI)


This Ansible role sets up backup scripts and systemd services to periodically archive specified directories as `.tar.gz` files and store them under a destination directory (default: `/var/backup/`).

The backup process uses a shell script (`dir_backup.sh`) to create compressed archives of target directories. The backup targets, output directory, schedule, and user/group are fully customizable via variables.

## How It Works

- The role installs backup scripts to the directory specified by `dir_backup_bin_dir` (default: `/opt/bin/`).
- The main backup script (`backup.sh`) iterates over the `dir_backup_targets` list and calls `dir_backup.sh` for each target.
- Each target directory is archived as a `.tar.gz` file and placed under the directory specified by `dir_backup_output_dir` (default: `/var/backup/<target_name>/`).
- Backup is scheduled via systemd timer (`backup.timer`) according to the `dir_backup_on_calender` variable.
- The backup script is downloaded from the repository specified by `dir_backup_core_script_repo`.

## User Requirements & Notes

- **You must define `dir_backup_targets`** in your inventory, group_vars, or playbook. Example:
  ```yaml
  dir_backup_targets:
    - name: nginx
      src: /usr/local/etc/nginx
    - name: etc
      src: /etc
    - name: opt
      src: /opt
  ```
- **Destination directory**: Set with `dir_backup_output_dir` (default: `/var/backup/`). Ensure sufficient disk space.
- **Backup script location**: Set with `dir_backup_bin_dir` (default: `/opt/bin/`).
- **Permissions**: Backup scripts run as the user/group specified in `dir_backup_user` and `dir_backup_group` (default: `root`). Make sure these have access to all target directories.
- **Schedule**: Set `dir_backup_on_calender` to control when backups run (systemd timer format, e.g. `*-*-* 01:00:00`).
- **Core script repo/version**: Set with `dir_backup_core_script_repo` and `dir_backup_core_script_version`.
- **Systemd**: The role creates and enables `backup.service` and `backup.timer`. Ensure systemd is available and enabled on your host.
- **Dry Run**: Use `--check` and `--diff` options with Ansible to preview changes.
- **Restore**: This role only creates backups. Restoration must be handled manually.
- **Customization**: All paths and variables can be overridden in your playbook or inventory.

## Example Variable Configuration
See `defaults/main.yml` for all available variables.

```yaml
dir_backup_targets:
  - name: nginx
    src: /usr/local/etc/nginx
  - name: etc
    src: /etc
  - name: opt
    src: /opt
dir_backup_output_dir: /var/backup
dir_backup_bin_dir: /opt/bin
dir_backup_on_calender: '*-*-* 01:00:00'
dir_backup_user: root
dir_backup_group: root
dir_backup_core_script_repo: https://github.com/Asya-kawai/dir-backup.git
dir_backup_core_script_version: HEAD
```

# Examples

## Directory Structure Example

```
.
├── README.md
├── ansible.cfg
├── group_vars
│   ├── all.yml
│   ├── webserver_centos.yml
│   └── webserver_ubuntu.yml
├── inventory
└── webservers.yml
```

## Inventory file

```
[webservers:children]
webserver_ubuntu
webserver_centos

[webserver_ubuntu]
ubuntu ansible_host=10.10.10.10 ansible_port=22

[webserver_centos]
centos ansible_host=10.10.10.11 ansible_port=22
```

## Group Vars / Common settings(all.yml)

`all.yml` sets common variables.

```
# Common settings
become: yes
ansible_user: root

# Private_key is saved in local host only!
ansible_ssh_private_key_file: ""
```

## Group Vars / Ubuntu(webserver_ubuntu.yml)

`webserver_ubuntu.yml` is `webservers` host's children.

This role references the `backup` variable,
The following example shows that files are backed up at 01:00:00 every day.

Note:

The format of `backup.on_calender` follows the `OnCalender` option of systemd service.

```
ansible_user: ubuntu
become: yes
ansible_become_password: 'ThisIsSecret!'

backup:
  user: root
  group: root
  on_calender: '*-*-* 01:00:00'
```

Of course, you can define it as a general setting in `all.yml`.

## Group Vars / CentOS(webserver_centos.yml)

`webserver_ubuntu.yml` is `webservers` host's children.

The following example shows that the backup user backs up files daily at 00:00:00.

```
# Use all.yml's settings.

backup:
  user: backup
  group: backup
  on_calender: '*-*-* 00:00:00'
```

## Playbook / Webservers(webservers.yml)

```
- hosts: webservers
  become: yes
  module_defaults:
    apt:
      cache_valid_time: 86400
  roles:
    - user
```

# How to DryRun and Apply

Dry Run

```
ansible-playbook -i inventory --private-key="~/.ssh/your_private_key" --check --diff webservers.yml --tags backup
```

Apply

```
ansible-playbook -i inventory --private-key="~/.ssh/your_private_key" webservers.yml --tags backup
```

## Troubleshooting & Tips

- If backups are not created, check systemd timer status (`systemctl status backup.timer`).
- Ensure all target directories exist and are readable by the backup user.
- Review logs in `/var/backup/` for errors.
- If `dir_backup.sh` is missing, verify internet connectivity and GitHub access.

## License & Author
See LICENSE for details. Role maintained by Asya-kawai.
