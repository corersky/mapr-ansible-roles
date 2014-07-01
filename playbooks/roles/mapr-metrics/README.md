Role Name
========

Installation of MapR metrics.

Requirements
------------

This playbook installs the MapR metrics package. This places metrics on hosts that are part of the webserver and jobtracker nodes.

Role Variables
--------------

None

Dependencies
------------

None

Example Playbook
-------------------------

```
---
- hosts: webserver:jobtracker
  sudo: yes
  roles:
    - mapr-metrics
```

License
-------

BSD