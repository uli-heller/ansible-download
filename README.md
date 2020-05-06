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
TASK [gitea : Download gitea.xz] *************************************************************************
fatal: [myohgserver.mydo...]: FAILED! => {"changed": false, "dest": "/tmp/gitea-1.11.4-linux-amd64.xz", 
  "elapsed": 0, "msg": "Request failed: <urlopen error [Errno -2] Name or service not known>",
  "url": "https://github.com/go-gitea/gitea/releases/download/v1.11.4/gitea-1.11.4-linux-amd64.xz"}

PLAY RECAP ***********************************************************************************************
myohgserver.mydo...: ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
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

Beobachtung: Ausführung klappt!

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
...
TASK [gitea : Download gitea.xz] *************************************************************************
changed: [myohgserver.mydomain.com -> localhost]

PLAY RECAP ***********************************************************************************************
myohgserver.mydo...: ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Task erneut ausführen: `ansible-playbook -i inventory.yml giteaservers.yml`

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
...
TASK [gitea : Download gitea.xz] *************************************************************************
ok: [myohgserver.mydomain.com -> localhost]

PLAY RECAP ***********************************************************************************************
myohgserver.mydo...: ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Beobachtung: Datei wird nicht erneut heruntergeladen - super!

Download und Upload
-------------------

Task erweitern um Upload auf den Server:

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
- name: Copy gitea.xz to {{inventory_hostname}}
  copy:
    src: "/tmp/gitea-1.11.4-linux-amd.xz"
    dest: /tmp/.
    mode: go-w
```

Tasks ausführen: `ansible-playbook -i inventory.yml giteaservers.yml`

```
$ ansible-playbook -i inventory.yml giteaservers.yml
PLAY [giteaservers] **************************************************************************************

TASK [Gathering Facts] ***********************************************************************************
ok: [myohgserver.mydomain.com]

TASK [gitea : Download gitea.xz] *************************************************************************
ok: [myohgserver.mydomain.com -> localhost]

TASK [gitea : Copy gitea.xz to myohgserver.mydomain.com] *************************************************
changed: [myohgserver.mydomain.com]

PLAY RECAP ***********************************************************************************************
myohgserver.mydo...: ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Erneute Ausführung: Kein Herunterladen, kein Hochladen!

Download/Upload nur wenn noch nicht vorhanden
---------------------------------------------

Der bisher erreichte Stand funktioniert im Wesentlichen.
Wenn ich das Playbook aber auf mehreren verschiedenen Rechnern ausführe,
dann wird GITEA jedesmal wieder heruntergeladen.

Wir wollen diese Änderung:

1. Prüfen, ob GITEA bereits auf dem Server vorhanden ist
2. Nur wenn noch nicht vorhanden: GITEA herunterladen und hochladen

Wir erreichen dies durch Erweitern der Tasks:

```diff
--- a/roles/gitea/tasks/main.yml
+++ b/roles/gitea/tasks/main.yml
@@ -1,5 +1,12 @@
 ---
 # tasks file for gitea - roles/gitea/tasks/main.yml
+- name: Check for gitea.xz on {{inventory_hostname}}
+  command:
+    argv:
+    - "echo"
+    - "/tmp/gitea-1.11.4-linux-amd.xz"
+    creates: "/tmp/gitea-1.11.4-linux-amd.xz"
+  register: gitea_check_for_giteaxz
 - name: Download gitea.xz
   get_url:
     url: https://github.com/go-gitea/gitea/releases/download/v1.11.4/gitea-1.11.4-linux-amd64.xz
@@ -7,8 +14,10 @@
   delegate_to: localhost
   vars:
     ansible_become: no
+  when: gitea_check_for_giteaxz.changed
 - name: Copy gitea.xz to {{inventory_hostname}}
   copy:
     src: "/tmp/gitea-1.11.4-linux-amd.xz"
     dest: /tmp/.
     mode: go-w
+  when: gitea_check_for_giteaxz.changed
```

Weitere Verbesserungen
----------------------

Für einen produktiven Einsatz sind u.a. noch diese Verbesserungen notwendig:

- Prüfen der Signaturen des Downloads
- Einsatz von Variablen

Probleme
--------

### myohgserver.mydomain.com: UNREACHABLE!

Beim Ausführen des Playbooks erscheint eine Fehlermeldung:

```
$ ansible-playbook -i inventory.yml giteaservers.yml 
PLAY [giteaservers] **************************************************************************************

TASK [Gathering Facts] ***********************************************************************************
fatal: [myohgserver.mydomain.com]: UNREACHABLE! => {"changed": false, "msg": 
 "Failed to connect to the host via ssh:
 ssh: Could not resolve hostname myohgserver.mydomain.com: Name or service not known",
 "unreachable": true}

PLAY RECAP ***********************************************************************************************
myohgserver.mydo...: ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
```

Abhilfe: Inventory anpassen!

Änderungen
----------

* 2020-05-06: Erste Version
