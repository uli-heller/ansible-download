Ansible: Download
=================

Ich möchte gerne GITEA aus dem Internet herunterladen
und auf einem meiner Server installieren.
Der Server hat keine Verbindung in's Internet.

<!-- more -->

Vorbereitungen
--------------

1. Ansible-Rolle anlegen "gitea": `(mkdir roles; cd roles; ansible-galaxy role init gitea)`
2. Playbook anlegen, welches diese Rolle zieht

```
---
# file: giteaservers.yml
- hosts: giteaservers
  serial: 1
  roles:
    - gitea
```

3. Inventory anlegen

```
---
# file: inventory.yml
all:
  hosts:
  children:
    giteaservers:
      hosts:
        myohgserver.mydomain.com
```

Erster Versuch
--------------

Task zum Runterladen einrichten:

```
---
# tasks file for gitea - roles/gitea/tasks/main.yml
- name: Download gitea.xz
  get_url:
    url: https://github.com/go-gitea/gitea/releases/download/v1.11.4/gitea-1.11.4-linux-amd64.xz
    dest: "/tmp/gitea-1.11.4-linux-amd64.xz"
```

Task ausführen: `ansible-playbook -i inventory.yml giteaservers.yml`

Beobachtung: Ausführung klappt nicht, runterladen scheitert!

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
...
TASK [gitea : Download gitea.xz] *********************************************************************************
fatal: [myohgserver.mydomain.com]: FAILED! => {"changed": false, "dest": "/tmp/gitea-1.11.4-linux-amd64.xz", 
  "elapsed": 0, "msg": "Request failed: <urlopen error [Errno -2] Name or service not known>",
  "url": "https://github.com/go-gitea/gitea/releases/download/v1.11.4/gitea-1.11.4-linux-amd64.xz"}

PLAY RECAP *******************************************************************************************************
myohgserver.mydomain.com   : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

Problem: Der Download wird von meinem Server angestossen, nicht von meinem lokalen Rechner!

Lokaler Download
----------------

Task zum Runterladen anpassen:

```
---
# tasks file for gitea - roles/gitea/tasks/main.yml
- name: Download gitea.xz
  get_url:
    url: https://github.com/go-gitea/gitea/releases/download/v1.11.4/gitea-1.11.4-linux-amd64.xz
    dest: "/tmp/gitea-1.11.4-linux-amd.xz"
  delegate_to: localhost
  vars:
    ansible_become: no
```

Task ausführen: `ansible-playbook -i inventory.yml giteaservers.yml`

Beobachtung: Ausführung klappt nicht, runterladen scheitert!

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
...
TASK [gitea : Download gitea.xz] *********************************************************************************
changed: [myohgserver.mydomain.com -> localhost]

PLAY RECAP *******************************************************************************************************
myohgserver.mydomain.com   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Download klappt!


Probleme
--------

### myohgserver.mydomain.com: UNREACHABLE!

Beim Ausführen des Playbooks erscheint eine Fehlermeldung:

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
PLAY [giteaservers] **********************************************************************************************

TASK [Gathering Facts] *******************************************************************************************
fatal: [myohgserver.mydomain.com]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh:
 ssh: Could not resolve hostname myohgserver.mydomain.com: Name or service not known", "unreachable": true}

PLAY RECAP *******************************************************************************************************
myohgserver.mydomain.com   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0   
```

Abhilfe: Inventory anpassen!

Änderungen
----------

* 2020-05-06: Erste Version