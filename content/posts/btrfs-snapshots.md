---
author: "Matic Rupnik"
authorLink: matic-rupnik
title: Handling BTRFS snapshots on Docker container volumes using Ansible
tags: [
  "Ansible",
  "BTRFS",
  "Docker"
]
date: 2023-04-04
toc: true
--- 

With the recent development of BTRFS filesystem along with its adaptation as a default filesystem for Linux distributions such as Fedora and and its copy-on-write(COW) principles it is becoming a system admins go-to filesystem for multiple things. In this short article I will be explaining an example of using the snapshotting capability of BTRFS with Ansible automation for backing up Docker volume of an application

To give some background. I self host my IT infrastructure for myself and family. In this case I needed a way to handle backups in case of breaking updates or general issues that come with the applications run(and trust me, there were some…). Since most of my applications are run as docker containers I needed to decide how to store the volumes to which I decided on running a BTRFS partition. This was an easy choice since it offers snapshotting out of the box.

Now to do backups and maintenance on the server in periodic intervals of say once a week, this became quite a nuisance to run. The simple premise revolves around the using the subvolume snapshot command on the location of the data you want to store and the path where you want to store it.

```bash
sudo btrfs subvolume snapshot /path/to/subvolume /path/to/snapshot
```
Lets presume that your docker volumes are stored on a BTRFS partition that is mounted on /mnt/data/<application_name> and you want a backup to be stored in /mnt/data/.snapshots/>application_name> this would be done with something like the following command.
```bash
sudo btrfs subvolume snapshot/mnt/data/<application_name> /mnt/data/.snapshots/<application_name>
```
Now to take it a step further we would automate this with a Ansible playbook.

```yaml
---
- name: Create btrfs snapshot of a Docker volume
  hosts: docker-host.lab
  become: true

  vars:
    snapshot_dir: /mnt/data/.snapshots/nextcloud
    source_dir: /mnt/data/nextcloud
    max_snapshots: 4

  tasks:
    - name: Stop Docker service
      systemd:
        name: docker.service
        state: stopped

    - name: Create btrfs snapshot
      command: btrfs subvolume snapshot {{ source_dir }} {{ snapshot_dir }}/{{ ansible_date_time.iso8601 }}
      changed_when: true

    - name: Delete oldest btrfs snapshot if more than max_snapshots exist
      shell: >
        ls -t {{ snapshot_dir }} |
        tail -n +{{ max_snapshots + 1 }} |
        xargs -I '{}' sudo rm -r {{ snapshot_dir }}/{{ ansible_date_time.iso8601 }}'{}'
      args:
        executable: /bin/bash
      register: delete_snapshot
      changed_when: delete_snapshot.stdout_lines | length > 0

    - name: Start Docker service
      systemd:
        name: docker.service
        state: started
```
To simply explain the above playbook will take the two locations that we mentioned above and a max snapshot number to create the snapshot. It will create a backup in the .snapshot directory(named as timestamp of creation) and than check if there are more than 4 snapshots already present, if there are it will delete the oldest one.

The inclusion of docker stop and start commands to prevent any data loss can admittedly be done a bit more gracefully but in an example scenario I have seen that this works fine.

To conclude, this is an easy example of how you can make your infrastructure backing up easier using Ansible and BTRFS filesystem.