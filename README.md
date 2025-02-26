# Testing Ansible Role With Molecule

Very basic example on how to test Ansible roles with Molecule in a podman or docker container.

## Basic Setup

- Install ansible and molecule

```
python3 -m venv ansiblevenv
source ansiblevenv/bin/activate
pip install molecule molecule-plugins[docker|podman] ansible pytest testinfra
```

- Create a role from templates

```
ansible-galaxy role init nginx
```

- Initialize molecule

```
molecule init scenario --driver-name docker|podman
```

- Update information in `nginx/meta/main.yml`

```
---
galaxy_info:
  role_name: nginx
  namespace: wrt
...
```

- Rename unneeded files from the molecule init step

```
mv -f nginx/molecule/default/create.yml nginx/molecule/default/create.yml.orig
mv -f nginx/molecule/default/destroy.yml nginx/molecule/default/destroy.yml.orig
```

These files are not needed for this basic test in a podman or docker container.

## Implement role and configure Molecule

- Implement role

- Configure Molecule

```
$ cat nginx/molecule/default/molecule.yml
---
driver:
  name: podman

platforms:
  - name: ubi9-container
    hostname: ubi9-container
    image: registry.access.redhat.com/ubi9/ubi
    privileged: true
    command: "/sbin/init"

provisioner:
  inventory:
    hosts:
      all:
        hosts:
          ubi9-container:
            ansible_python_interpreter: /usr/bin/python3
  name: ansible

verifier:
  name: testinfra|ansible
```

Depending on how the role should be tested, the `verifier` directive needs to be set accordingly.
More information can be found in the next subsections.

- Implement converge playbook, it installs the role

```
$ cat nginx/molecule/default/converge.yml
---
- name: Verify Install Nginx Role
  hosts: all
  roles:
    - nginx
```

### Testing with `testinfra`

When testing with `testinfra`, Python is used for testing inside the container.

First the verifier should be set to `testinfra`:

```
$ cat nginx/molecule/default/molecule.yml
[..]
verifier:
  name: testinfra
```

The tests are written in `.py` files inside the `nginx/molecule/default/tests` directory.
Here an example:

```
$ cat nginx/molecule/default/tests/test_default.py
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')

# def test_user(host):
#     user = host.user("www-data")
#     assert user.exists

def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed

def test_nginx_running_and_enabled(host):
    nginx1 = host.service("nginx")
    assert nginx1.is_running
    assert nginx1.is_enabled

def test_nginx_is_listening(host):
    assert host.socket('tcp://127.0.0.1:80').is_listening
```

All `.py` files in `nginx/molecule/default/tests` will be run as tests.

### Testing with `ansible`

When testing with Ansible, Ansible playbooks are being used.

First the verifier should be set to `ansible`:

```
$ cat nginx/molecule/default/molecule.yml
[..]
verifier:
  name: ansible
```

Creating a new role with the `ansible-galaxy role init` command, all default directories are being created.
So is the `tests` directory created, with a `test.yml` playbook file and a test `inventory` file.

To use this testing playbook in molecule, we can import it in the verify step:

```
$ cat nginx/molecule/default/verify.yml
---
- name: Run Ansible role tests
  import_playbook: ../../tests/test.yml
```

A playbook to test this example nginx role could look like the following:

```
$ cat nginx/tests/test.yml
---
- name: Verify Nginx Installation
  hosts: all
  gather_facts: no
  tasks:

    - name: Check if Nginx package is installed
      ansible.builtin.package:
        name: nginx
        state: present
      register: nginx_package
      failed_when: nginx_package is failed

    - name: Ensure Nginx service is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
      register: nginx_service
      failed_when: not nginx_service.status.ActiveState == "active"
```

### Run commands inside the container to prepare it for the test (optional)

When using a standard container image it might be necessary to include additional preparation steps on the container.
This can be done through the optional prepare playbook.

```
$ cat nginx/molecule/default/molecule.yml
...
provisioner:
  ...
  name: ansible
  log: true
  playbooks:
    prepare: ../prepare.yml
```

This will run the playbook in the prepare.yml:

```
---
- name: Prepare test container
  hosts: all
  tasks:
    - name: Install dependencies for testing
      ansible.builtin.package:
        name:
          - python3-pip
          - python3
        state: present

    - name: Install pytest and testinfra
      ansible.builtin.pip:
        name:
          - pytest
          - testinfra
```

## Molecule stages

- Run all stages and test the role

```
molecule test
```

- Create the environment

```
molecule create
```

This will create a podman container.

```
(ansvenv) [wreiner@localhost nginx_role]$ podman ps
CONTAINER ID  IMAGE                                                                COMMAND     CREATED         STATUS         PORTS       NAMES
1b4fd01a99b4  localhost/molecule_local/registry.access.redhat.com/ubi9/ubi:latest  /sbin/init  31 seconds ago  Up 31 seconds              ubi9-container
```

- Create the environment and run the role

```
molecule converge
```

The converge playbook will be run a second time to check for idempotency.

- Perform the actual tests

```
molecule verify
```

- Destroy the environment

```
molecule destroy
```

This will stop and delete the podman container.

```
(ansvenv) [wreiner@localhost nginx_role]$ podman ps
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

- Login to testing environment

```
molecule login
```

Or use podman|docker to exec into the container.

## Linting

You should run linting. It was removed from molecule and should be run seperately in CI/CD.
- [https://github.com/ansible/molecule/discussions/3914]
- [https://github.com/ansible/molecule/pull/3802]
