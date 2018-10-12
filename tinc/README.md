# tinc

Example playbook for deploying a mesh network on a number of hosts.

---

There are numerous ways this can be achieved, but for the sake of simplicity each node will be identified via the last octet of it's public IP address. So, for example, if you're using the `10.10.1.0/32` subnet and the node is reachable from the internet on `xxx.xxx.xxx.289` then it will also be reachable via `10.10.1.289` on the VPN.

Things to be aware of:

- Each host must have a unique hostname - this is used when setting up tinc node. No worries if they're not, ansible will warn you when it's not the case
- Make sure port 655 can be reached on TCP and UDP - tinc uses them to establish a connection
- By default, a mesh network with a name of `mytinc` and a subnet `10.10.1.0/32` will be created
- tinc daemon can be started/stopped: `service tinc@mytinc start`

## How does it work?

Each playbook run generates new SSH keys for every node, so it can also be easily used to rotate SSH keys. Generally it works like this:

- Create temporary build folder
- Provision each node
- Synchronize host configurations
- Delete build folder

**Create temporary build folder** - will keep all SSH keys and daemon configs, so that later we can distribute public keys to all hosts.

**Provision each node** - installs tinc server, uploads daemon configuration files and private SSH key used to secure communication between the nodes.

**Synchronize host configurations** - uploads tinc host configuration files to all nodes. this file has the node ip, subnet configuration and it’s public key.

**Delete build folder** - deletes temporary build root, removing all traces of generated SSH keys.

Example playbook:

```yml
---
# Deploys tinc VPN

- name: Create build directory
  hosts: 127.0.0.1
  connection: local
  gather_facts: false
  tasks:
    - name: Create build directory
      tempfile: state=directory suffix=_tinc
      register: build_dir
    - name: Make it available to other plays
      set_fact: build_dir="{{ build_dir.path }}"
    - debug:
        msg: "Build root: {{ build_dir }}"

- name: Provision tinc server
  hosts: all
  vars:
    tinc_build_root: "{{ hostvars['localhost']['build_dir'] }}"
  roles:
    - tinc

- name: Cleanup
  hosts: 127.0.0.1
  connection: local
  gather_facts: false
  tasks:
    - name: Delete the build folder
      file: state=absent path="{{ hostvars['localhost']['build_dir'] }}/"
```

More configuration options and explanations in the [defaults/main.yml](/tinc/defaults/main.yml)
