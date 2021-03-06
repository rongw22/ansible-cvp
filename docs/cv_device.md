# Manage device content

- [Manage device content](#manage-device-content)
  - [Description](#description)
  - [Options](#options)
  - [Usage](#usage)
    - [Authentication](#authentication)
    - [Inputs example](#inputs-example)
    - [Result example](#result-example)
  - [Use cases](#use-cases)
    - [Reset devices to undefined container](#reset-devices-to-undefined-container)

## Description

__Module name:__ `arista.cvp.cv_device`

This module manage devices from a Cloudvision stand point. It takes an intended list of devices with their configlets name and image bundle, compare against facts from [`cv_facts`](cv_facts.md) and then __attach__, __remove__ both configlets and image bundles.

## Options

Module comes with a set of options:

- `devices`: List of devices to manage.
- `cvp_facts`: Current facts collecting on CVP by a previous task
- `devices_filter`: Filter to apply intended mode on a set of configlet. If not used, then module only uses ADD mode. device_filter list devices that can be modified or deleted based on configlets entries.
- `state`: . Provide an option to delete devices from topology and reset devices to undefined container. Default value is `present` and it is an optional field.
  - `absent`: Reset devices.
  - `present`: Configure devices.
- `configlet_fitler`: Allow user to select behaviour to apply on configlets attached to a device:
  - `override`: (default) Configure list provided by user and remove any other configlets attached to device.
  - `merge`: add configlets listed by user to the device and do not touch existing configlets.
  - `delete`: Remove list of configlets from device and let other unchanged.

## Usage

### Authentication

This module uses `HTTPAPI` connection plugin for authentication. These elements shall be declared using this plugin mechanism and are automatically shared with `arista.cvp.cv_*` modules.

```ini
[development]
cvp_foster  ansible_httpapi_host=10.90.224.122

[development:vars]
ansible_connection=httpapi
ansible_httpapi_use_ssl=True
ansible_httpapi_validate_certs=False
ansible_user=cvpadmin
ansible_password=ansible
ansible_network_os=eos
ansible_httpapi_port=443
```

### Inputs example

Below is a basic playbook to collect facts:

```yaml
  vars:
    devices_inventory:
      veos01:
        name: veos01
        configlets:
          - cv_device_test01
          - SYS_TelemetryBuilderV2_172.23.0.2_1
          - veos01-basic-configuration
          - SYS_TelemetryBuilderV2
        parentContainerName: DC1_VEOS
        imageBundle: []
  tasks:
    - name: 'Collecting facts from CVP {{inventory_hostname}}.'
      arista.cvp.cv_facts:
      register: cvp_facts

    - name: "Configure devices on {{inventory_hostname}}"
      arista.cvp.cv_device:
        devices: "{{devices_inventory}}"
        cvp_facts: '{{cvp_facts.ansible_facts}}'
        device_filter: ['veos']
        state: present
      register: cvp_device
```

### Result example

Below is an example of expected output

```json
{
    "changed": "True",
    "data": {
        "tasks": [
            {
                "workOrderId": "108",
                "name": "",
                "workOrderState": "ACTIVE"
            }
        ],
        "added_tasksIds": [
            "118"
        ],
        "provisionned": [
            {
                "veos-01": "provisionning-tasks-['118']"
            }
        ],
        "provisionned_devices": 1,
        "updated_devices": 1,
        "updated_tasksIds": [
            "108"
        ],
        "updated": [
            {
                "DC1-SPINE1": "Configlets-['108']"
            }
        ],
        "tasksIds": [
            "108"
        ]
    }
}
```

## Use cases

### Reset devices to undefined container

Whit this example, `veos01` device will be reset with initial configuration (ZTP mode) and move back to `undefined` container.

```yaml
---
- name: Build Testing topology using Dev CVP servers
  hosts: cvp
  connection: local
  gather_facts: no
  vars:
    DEVEL_CONTAINERS:
      staging:
        parent_container: Tenant
    DEVEL_DEVICES:
      veos01:
        name: veos01
        configlets:
          - DEV_VEOS01
        imageBundle: []
        parentContainerName: staging
  tasks:
    - name: 'Collect facts from {{inventory_hostname}}'
      cv_facts:
      register: facts

    - name: 'Reset devices on {{inventory_hostname}}'
      cv_device:
        devices: '{{DEVEL_DEVICES}}'
        cvp_facts: '{{facts.ansible_facts}}'
        device_filter: ['veos01']
        state: absent
      register: devices_result

    - name: 'Run tasks'
      cv_task:
        tasks: '{{devices_result.data.tasks}}'
        # Wait 7 minutes for device to reboot
        wait: 420
```

> In this scenario, tasks are returned by CVP only if we run reset. If you run playbook with no task execution and then rerun same playbook to execute task, CVP won't return it.
