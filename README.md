Linux Update Role
-----------------

This role is used to update packages/kernels for Linux (CentOS 8) servers.

# Functionalities

 * Send status messages to a chat application, also alert when something failed
 * Set downtime in a monitoring tool
 * Set downtime for hosts in loadbalancers
 * Saves a list of packages before and after the updates
 * Include user defined playbooks, e.g. when a cluster requires a rolling restart
 * Reboot hosts and verifies SSH connection
 * Ensure required applications are started
 * Package exclusion for updates
 * Health check specific URLs

## Version info

 * 0.1 - initial release
 * 0.2 - added yamllint and ansible-lint and improved json output
 * 0.3 - added molecule testing

## Info about installed packages

By default, a pre and post list is retrieved containing information about the installed packages per host.
The list is saved on the Ansible Master and can be used for comparison.

A folder is created of the current day and the files are saved in that folder, under the user running the ansible-playbook:

```bash
/var/linux_updates/2021-01-01/p-web-01/pre_packages.json
/var/linux_updates/2021-01-01/p-web-02/post_packages.json
/var/linux_updates/2021-01-01/p-web-02/pre_packages.json
/var/linux_updates/2021-01-01/p-web-02/post_packages.json
```

The output is saved in JSON format, apply filtering with `jq`. Find e.g. specific detailed package information with:

```bash
>  cat pre_packages.json | jq .zlib
[
  {
    "arch": "x86_64",
    "epoch": null,
    "name": "zlib",
    "release": "16.el8_2",
    "source": "rpm",
    "version": "1.2.11"
  }
]
```

Specify your desired path with:

```yaml
# Do not include the final '/' in the path
linux_update_package_info_path: /var/linux_updates
```

Note that the content saved should not be used as a source of truth for installed packages. But rather a reference to quickly see what was updated/installed before and after the update.

## Include playbooks and/or commands

It's possible to include user defined playbooks during execution. This is useable for e.g. rolling restarts.

The user defined commands are executed after the include playbook, and are executed via the `shell` module.
```yaml
linux_update_pre_cmd:
  - mysqld configtest -f /etc/mysql/mysqld.cnf
  - php clean cache
linux_update_post_cmd:
  - /bin/true
  - mysqld configtest -f /etc/mysql/mysqld.cnf

linux_update_pre_tasks_playbook: /path/to/my_pre_task_playbook.yml
linux_update_pre_reboot_playbook: /path/to/my_pre_reboot_playbook.yml
linux_update_post_reboot_playbook: /path/to/my_post_reboot_playbook.yml
linux_update_post_tasks_playbook: /path/to/my_post_tasks_playbook.yml
```

Specify the paths of the playbooks in the host or group vars of the host.
The playbook itself has to be present on the Ansible Master, which executes the playbook.

Example of an include playbook:

```yaml
---
- name: do something
  shell: 'echo something'
  register: the_echo

- debug:
    msg: "{{ the_echo.stdout }}"
```

## Reboot

If you do not want to restart the server when there are no package changed, set:

```yaml
linux_update_reboot: false
```

By default this is set to `true`.

## Applications

You can specify applications, which are stopped before updating. Then, after the reboot, they are ensured to be started again, in reverse order.

```yaml
linux_update_applications:
  - mysqld
  - httpd
  - haproxy
```

Errors are ignored when stopping the services.

## Exclude package

Yum has an [exclude option](https://docs.ansible.com/ansible/latest/modules/yum_module.html#parameter-exclude). For Ubuntu, Ansible ['holds' the packages](https://github.com/ansible/ansible-modules-core/issues/19).

```yaml
linux_update_excl:
  - nano
  - mysql
```

## Old kernels

Old kernels are removed. By default the newest kernel is kept.

```yaml
linux_update_old_kernels: 1
```

## Default repository files

CentOS places the default repo files on the system [after an upgrade](centos-release). Deleting these files don't have the desired effect, as they will be re-created. Thus, we `touch` the files, so at least they're present on the system.

## Health check

The health check is executed before the update and after the update of the packages.
By default, certificate validation is turned off.

```yaml
linux_update_health_check_url: www.my_site.com/page.html
linux_update_health_check_valide_cert: no
```

The URL can also be an internal address, e.g. `10.0.0.1/index.html`.

The role checks whether status 200 is provided. A more detailed or fine grained health check should be configured into the monitoring system.

## Example playbook

An example playbook when updating several web servers:

```yaml
---
- hosts:
    - p-web-01
    - p-web-02
    - p-web-03
    - p-web-04
  become: true
  any_errors_fatal: false
  roles:
    - linux_updates
```

An example playbook when updating a database cluster:

```yaml
---
- hosts:
    - p-db-01
    - p-db-02
  become: true
  any_errors_fatal: true
  serial: 1
  roles:
    - linux_updates
```
