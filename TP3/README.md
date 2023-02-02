
Notre setup.yml : 

```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /fs03/share/users/mohamed.benyoub/home/Documents/S8DevOps/TP3/ansible/inventories/id_rsa
 children:
   prod:
     hosts: mohamed.benyoub.takima.cloud
```
Résultat : 
``` bash
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible all -i inventories/setup.yml -m ping
mohamed.benyoub.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    }, 
    "changed": false, 
    "ping": "pong"
}
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"

mohamed.benyoub.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS", 
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "8", 
        "ansible_distribution_release": "NA", 
        "ansible_distribution_version": "8", 
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    }, 
    "changed": false
}
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ 
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become

mohamed.benyoub.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    }, 
    "changed": false, 
    "msg": "Nothing to do", 
    "rc": 0, 
    "results": []
}
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ 
```
Notre playbook.yml
```yml
- hosts: all
  gather_facts: false
  become: yes

  tasks:
   - name: Test connection
     ping:
```
Résultat : 
```bash
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-playbook -i inventories/setup.yml playbook.yml

PLAY [all] *************************************************************************************************************************************************************

TASK [Test connection] *************************************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

PLAY RECAP *************************************************************************************************************************************************************
mohamed.benyoub.takima.cloud : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
Changement de notre playbook puis relance le commande précédente : 
``` bash
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-playbook -i inventories/setup.yml playbook.yml

PLAY [all] *************************************************************************************************************************************************************

TASK [Clean packages] **************************************************************************************************************************************************
[WARNING]: Consider using the dnf module rather than running 'dnf'.  If you need to use command because dnf is insufficient you can add 'warn: false' to this command
task or set 'command_warnings=False' in ansible.cfg to get rid of this message.
changed: [mohamed.benyoub.takima.cloud]

TASK [Install device-mapper-persistent-data] ***************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [Install lvm2] ****************************************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [add repo docker] *************************************************************************************************************************************************
[WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo
changed: [mohamed.benyoub.takima.cloud]

TASK [Install Docker] **************************************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [install python3] *************************************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [Pip install] *****************************************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [Make sure Docker is running] *************************************************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

PLAY RECAP *************************************************************************************************************************************************************
mohamed.benyoub.takima.cloud : ok=8    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

```
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/docker

- Role roles/docker was created successfully
```

Ensuite on crée tout les roles dont on a besoin : 

```
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/docker_create_network
- Role roles/docker_create_network was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_database
- Role roles/launch_database was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_app
- Role roles/launch_app was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_proxy
- Role roles/launch_proxy was created successfully
```