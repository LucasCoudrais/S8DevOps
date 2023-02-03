# Set-up Ansible
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
```
On arrive bien à se connecter et à intéragir avec notre centOS AWS grace à ansible
# Playbook
## Test de connexion
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
## Première installation de docker
Notre playbook.yml
```yml
- hosts: all
  gather_facts: false
  become: yes

# Install Docker
  tasks:
  - name: Clean packages
    command:
      cmd: dnf clean -y packages

  - name: Install device-mapper-persistent-data
    dnf:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    dnf:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    dnf:
      name: docker-ce
      state: present

  - name: install python3
    dnf:
      name: python3

  - name: Pip install
    pip:
      name: docker

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```
On exécute une liste de tache donnée pour installer docker sur notre centOS.

Résultat : 
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
Chaque ligne représente l'exécution d'une tache, si la tache à déjé été faite le statut sera à `ok` et pas `changed` ainsi, ansible va optimiser et ne pas relancer des taches déjà exécuté, sauf si elles ont changé. 
Maintenant le but est d'installer notre appli sur notre cent os entièrement avec ansible. Ainsi nous allons devoir rajouter un bon nombre de tache consistant à : 
- intaller docker
- créer un réseau pour que nos conteneur puissent communiquer
- installer la base de donnée depuis notre image sur notre docker hub
- installer de la meme manière notre backend 
- installer notre httpd 


## Découpage de nos taches en roles
```
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/docker
- Role roles/docker was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/docker_create_network
- Role roles/docker_create_network was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_database
- Role roles/launch_database was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_app
- Role roles/launch_app was created successfully
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-galaxy init roles/launch_proxy
- Role roles/launch_proxy was created successfully
```
Puis, dans chaque role nous allons exéctuer des taches ansible pour faire ce que l'on veut. Ensuite dans notre `playbook.yml` nous allons appeller l'ensemble de nos role dans un ordre voulu. Le but étant que à l'éxécution de notre playbook : nous allons déployer nos conteneur docker à partir de nos image sur docker hub du TP d'avant directement sur notre instance centOS
```
Voir le fichier playbook.yml et tous les main.yml qui se situent dans chaque dossier tasks de tous nos roles
```
Avec nos fichiers, nous obtenons donc le résultat suivant à l'éxécution de notre playbook :
```
mohamed.benyoub@tpc30:~/Documents/S8DevOps/TP3/ansible$ ansible-playbook -i inventories/setup.yml playbook.yml

PLAY [all] *************************************************************************************************************************************************************

TASK [docker : Install device-mapper-persistent-data] ******************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [docker : Install lvm2] *******************************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [add repo docker] *************************************************************************************************************************************************
[WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo
changed: [mohamed.benyoub.takima.cloud]

TASK [docker : Install Docker] *****************************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [docker : install python3] ****************************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [docker : Pip install] ********************************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [docker : Make sure Docker is running] ****************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [docker_create_network : Create a network] ************************************************************************************************************************
ok: [mohamed.benyoub.takima.cloud]

TASK [launch_database : Create db container and connect to network] ****************************************************************************************************
[DEPRECATION WARNING]: Please note that docker_container handles networks slightly different than docker CLI. If you specify networks, the default network will still 
be attached as the first network. (You can specify purge_networks to remove all networks not explicitly listed.) This behavior will change in Ansible 2.12. You can 
change the behavior now by setting the new `networks_cli_compatible` option to `yes`, and remove this warning by setting it to `no`. This feature will be removed in 
version 2.12. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
ok: [mohamed.benyoub.takima.cloud]

TASK [launch_app : Create app container and connect to network] ********************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

TASK [launch_proxy : Create proxy container and connect to network] ****************************************************************************************************
changed: [mohamed.benyoub.takima.cloud]

PLAY RECAP *************************************************************************************************************************************************************
mohamed.benyoub.takima.cloud : ok=11   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

Puis nous pouvons bien voir que nous avons accès à notre API à travers notre reverse proxy qui est bien lié a notre BDD en allant sur le domaine de notre instance centOS : 
![alt text](/TP3/img/Capture%20d%E2%80%99%C3%A9cran%20de%202023-02-02%2017-49-28.png)
