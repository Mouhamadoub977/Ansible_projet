# Tutoriel Ansible pour Débutants

Ce tutoriel est conçu pour les personnes qui découvrent Ansible et souhaitent apprendre à automatiser le déploiement d'applications et la configuration de serveurs.

## Table des matières

1. [Introduction à Ansible](#1-introduction-à-ansible)
2. [Installation d'Ansible](#2-installation-dansible)
3. [Structure d'un projet Ansible](#3-structure-dun-projet-ansible)
4. [Inventaire](#4-inventaire)
5. [Playbooks](#5-playbooks)
6. [Rôles](#6-rôles)
7. [Variables](#7-variables)
8. [Templates](#8-templates)
9. [Handlers](#9-handlers)
10. [Conditions et boucles](#10-conditions-et-boucles)
11. [Exemple pratique : Déploiement d'Apache2 et Jenkins](#11-exemple-pratique--déploiement-dapache2-et-jenkins)
12. [Dépannage](#12-dépannage)
13. [Bonnes pratiques](#13-bonnes-pratiques)
14. [Ressources supplémentaires](#14-ressources-supplémentaires)

## 1. Introduction à Ansible

### Qu'est-ce qu'Ansible ?

Ansible est un outil d'automatisation open-source qui permet de :
- Configurer des systèmes
- Déployer des applications
- Orchestrer des tâches plus complexes comme des déploiements continus ou des mises à jour sans temps d'arrêt

### Pourquoi utiliser Ansible ?

- **Sans agent** : Aucun logiciel à installer sur les serveurs cibles (utilise SSH)
- **Simple** : Utilise YAML, un langage facile à lire et à écrire
- **Idempotent** : Exécuter les mêmes tâches plusieurs fois ne change pas le résultat
- **Extensible** : Des milliers de modules disponibles pour différentes tâches

### Comment fonctionne Ansible ?

1. Ansible se connecte aux serveurs cibles via SSH
2. Envoie des "modules" (petits programmes) aux serveurs
3. Exécute ces modules et récupère les résultats
4. Continue avec les tâches suivantes en fonction des résultats

## 2. Installation d'Ansible

### Sur Ubuntu/Debian

```bash
sudo apt update
sudo apt install ansible
```

### Sur macOS

```bash
brew install ansible
```

### Vérification de l'installation

```bash
ansible --version
```

## 3. Structure d'un projet Ansible

Un projet Ansible typique est organisé comme suit :

```
mon_projet_ansible/
├── ansible.cfg           # Configuration Ansible
├── inventory/            # Inventaire des hôtes
│   └── hosts             # Fichier d'inventaire principal
├── group_vars/           # Variables pour des groupes d'hôtes
│   └── all.yml           # Variables pour tous les hôtes
├── host_vars/            # Variables pour des hôtes spécifiques
│   └── serveur1.yml      # Variables pour serveur1
├── roles/                # Rôles Ansible
│   ├── common/           # Rôle "common"
│   │   ├── tasks/        # Tâches du rôle
│   │   ├── handlers/     # Handlers du rôle
│   │   ├── templates/    # Templates du rôle
│   │   ├── files/        # Fichiers statiques du rôle
│   │   ├── vars/         # Variables du rôle
│   │   └── defaults/     # Variables par défaut du rôle
│   └── webserver/        # Autre rôle
├── playbook.yml          # Playbook principal
└── README.md             # Documentation
```

## 4. Inventaire

L'inventaire est la liste des serveurs qu'Ansible va gérer. Il peut être statique (fichier) ou dynamique (script).

### Exemple d'inventaire statique (inventory/hosts)

```ini
# Groupe de serveurs web
[webservers]
web1 ansible_host=192.168.1.101 ansible_user=ubuntu
web2 ansible_host=192.168.1.102 ansible_user=ubuntu

# Groupe de serveurs de base de données
[dbservers]
db1 ansible_host=192.168.1.201 ansible_user=ubuntu

# Groupe qui contient d'autres groupes
[production:children]
webservers
dbservers

# Variables pour tous les serveurs de production
[production:vars]
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### Paramètres courants dans l'inventaire

- `ansible_host` : Adresse IP ou nom d'hôte
- `ansible_user` : Utilisateur pour la connexion SSH
- `ansible_port` : Port SSH (par défaut 22)
- `ansible_ssh_private_key_file` : Chemin vers la clé privée SSH

### Test de l'inventaire

```bash
# Liste tous les hôtes
ansible all --list-hosts

# Ping tous les hôtes
ansible all -m ping
```

## 5. Playbooks

Les playbooks sont des fichiers YAML qui décrivent les tâches à exécuter sur les serveurs cibles.

### Structure d'un playbook

```yaml
---
- name: Configuration des serveurs web
  hosts: webservers
  become: yes  # Utilise sudo
  vars:
    http_port: 80
  
  tasks:
    - name: Installation d'Apache
      apt:
        name: apache2
        state: present
    
    - name: Démarrage du service Apache
      service:
        name: apache2
        state: started
        enabled: yes
```

### Exécution d'un playbook

```bash
ansible-playbook playbook.yml
```

### Options utiles

```bash
# Exécution en mode simulation (dry-run)
ansible-playbook playbook.yml --check

# Exécution sur des hôtes spécifiques
ansible-playbook playbook.yml --limit web1

# Exécution avec des variables supplémentaires
ansible-playbook playbook.yml --extra-vars "http_port=8080"
```

## 6. Rôles

Les rôles permettent d'organiser les playbooks en unités réutilisables.

### Structure d'un rôle

```
roles/webserver/
├── tasks/
│   └── main.yml          # Tâches principales
├── handlers/
│   └── main.yml          # Handlers (actions déclenchées par des notifications)
├── templates/
│   └── vhost.conf.j2     # Templates Jinja2
├── files/
│   └── app.conf          # Fichiers statiques
├── vars/
│   └── main.yml          # Variables du rôle
└── defaults/
    └── main.yml          # Variables par défaut (priorité la plus basse)
```

### Création d'un rôle

```bash
ansible-galaxy init roles/webserver
```

### Utilisation d'un rôle dans un playbook

```yaml
---
- name: Configuration des serveurs web
  hosts: webservers
  become: yes
  
  roles:
    - webserver
```

## 7. Variables

Les variables permettent de rendre les playbooks plus flexibles et réutilisables.

### Définition de variables

```yaml
# Dans un playbook
vars:
  http_port: 80
  max_clients: 200

# Dans un fichier séparé (group_vars/webservers.yml)
---
http_port: 80
max_clients: 200
```

### Utilisation de variables

```yaml
tasks:
  - name: Configuration d'Apache
    template:
      src: httpd.conf.j2
      dest: /etc/httpd/conf/httpd.conf
    vars:
      port: "{{ http_port }}"
```

### Précédence des variables

De la priorité la plus basse à la plus haute :
1. Variables par défaut du rôle (roles/x/defaults/main.yml)
2. Variables d'inventaire (group_vars/all.yml)
3. Variables d'inventaire pour le groupe (group_vars/groupname.yml)
4. Variables d'inventaire pour l'hôte (host_vars/hostname.yml)
5. Variables du playbook (vars: section)
6. Variables de ligne de commande (--extra-vars)

## 8. Templates

Les templates utilisent le moteur Jinja2 pour générer des fichiers de configuration dynamiques.

### Exemple de template (templates/vhost.conf.j2)

```jinja
<VirtualHost *:{{ http_port }}>
    ServerAdmin webmaster@{{ domain }}
    ServerName {{ domain }}
    DocumentRoot /var/www/{{ domain }}/public_html
    
    {% if ssl_enabled %}
    SSLEngine on
    SSLCertificateFile {{ ssl_cert_path }}
    {% endif %}
    
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Utilisation d'un template

```yaml
- name: Configuration du vhost Apache
  template:
    src: vhost.conf.j2
    dest: /etc/apache2/sites-available/{{ domain }}.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart Apache
```

## 9. Handlers

Les handlers sont des tâches qui ne s'exécutent que lorsqu'elles sont notifiées par d'autres tâches.

### Définition de handlers

```yaml
# Dans handlers/main.yml
---
- name: Restart Apache
  service:
    name: apache2
    state: restarted

- name: Reload Apache
  service:
    name: apache2
    state: reloaded
```

### Notification d'un handler

```yaml
- name: Configuration d'Apache
  template:
    src: apache.conf.j2
    dest: /etc/apache2/apache2.conf
  notify: Restart Apache
```

## 10. Conditions et boucles

### Conditions

```yaml
- name: Installation de packages spécifiques à Debian
  apt:
    name: apache2
    state: present
  when: ansible_os_family == "Debian"

- name: Installation de packages spécifiques à RedHat
  yum:
    name: httpd
    state: present
  when: ansible_os_family == "RedHat"
```

### Boucles

```yaml
- name: Installation de plusieurs packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - apache2
    - php
    - mysql-server
    - python3-pip

# Avec des dictionnaires
- name: Création d'utilisateurs
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop:
    - { name: 'john', groups: 'admin' }
    - { name: 'jane', groups: 'dev', shell: '/bin/zsh' }
```

## 11. Exemple pratique : Déploiement d'Apache2 et Jenkins

Voici un exemple complet pour déployer Apache2 et Jenkins sur des serveurs Ubuntu.

### Structure du projet

```
projet_apache_jenkins/
├── ansible.cfg
├── inventory/
│   └── hosts
├── roles/
│   ├── apache2/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── templates/
│   │       └── index.html.j2
│   └── jenkins/
│       └── tasks/
│           └── main.yml
├── site.yml
└── fix_jenkins_java_path.yml
```

### ansible.cfg

```ini
[defaults]
inventory = inventory/hosts
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/.ssh/ma_cle.pem
roles_path = roles
```

### inventory/hosts

```ini
[webservers]
host1 ansible_host=192.168.1.101 ansible_user=ubuntu
host2 ansible_host=192.168.1.102 ansible_user=ubuntu

[jenkins]
host1 ansible_host=192.168.1.101 ansible_user=ubuntu
```

### site.yml

```yaml
---
- name: Deploy Apache2 on webservers
  hosts: webservers
  become: yes
  vars:
    apache_port: 80
  
  tasks:
    - name: Set host-specific variables
      set_fact:
        hello_message: "Hello World 1"
      when: inventory_hostname == "host1"
    
    - name: Set host-specific variables
      set_fact:
        hello_message: "Hello World 2"
      when: inventory_hostname == "host2"
  
  roles:
    - apache2

- name: Deploy Jenkins on host1
  hosts: jenkins
  become: yes
  
  roles:
    - jenkins
```

### roles/apache2/tasks/main.yml

```yaml
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install Apache2
  apt:
    name: apache2
    state: present

- name: Ensure Apache2 service is running and enabled
  service:
    name: apache2
    state: started
    enabled: yes

- name: Create custom index.html with dynamic content
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: www-data
    group: www-data
    mode: '0644'
  notify: Restart Apache
```

### roles/apache2/handlers/main.yml

```yaml
---
- name: Restart Apache
  service:
    name: apache2
    state: restarted
```

### roles/apache2/templates/index.html.j2

```html
<!DOCTYPE html>
<html>
<head>
    <title>Hello World</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            color: #333;
            text-align: center;
            padding: 50px;
        }
        h1 {
            color: #0066cc;
            font-size: 3em;
        }
    </style>
</head>
<body>
    <h1>{{ hello_message }}</h1>
    <p>Serveur: {{ ansible_hostname }}</p>
</body>
</html>
```

### roles/jenkins/tasks/main.yml

```yaml
---
- name: Install dependencies
  apt:
    name:
      - openjdk-17-jdk
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present
    update_cache: yes

- name: Add Jenkins apt key
  apt_key:
    url: https://pkg.jenkins.io/debian-stable/jenkins.io.key
    state: present

- name: Add Jenkins apt repository
  apt_repository:
    repo: deb https://pkg.jenkins.io/debian-stable binary/
    state: present
    filename: jenkins

- name: Update apt cache
  apt:
    update_cache: yes

- name: Install Jenkins
  apt:
    name: jenkins
    state: present

- name: Ensure Jenkins service is running and enabled
  service:
    name: jenkins
    state: started
    enabled: yes

- name: Wait for Jenkins to start up
  wait_for:
    port: 8080
    delay: 10
    timeout: 300

- name: Get Jenkins initial admin password
  command: cat /var/lib/jenkins/secrets/initialAdminPassword
  register: jenkins_admin_password
  changed_when: false

- name: Display Jenkins initial admin password
  debug:
    msg: "Jenkins initial admin password: {{ jenkins_admin_password.stdout }}"
```

### fix_jenkins_java_path.yml

```yaml
---
- name: Configure Jenkins to use Java 17 explicitly
  hosts: jenkins
  become: yes
  
  tasks:
    - name: Ensure Jenkins configuration directory exists
      file:
        path: /etc/systemd/system/jenkins.service.d
        state: directory
        mode: '0755'
        
    - name: Create Jenkins override configuration
      copy:
        dest: /etc/systemd/system/jenkins.service.d/override.conf
        content: |
          [Service]
          Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
          Environment="JENKINS_JAVA_CMD=/usr/lib/jvm/java-17-openjdk-amd64/bin/java"
        mode: '0644'
      register: jenkins_override
        
    - name: Update Jenkins default configuration
      lineinfile:
        path: /etc/default/jenkins
        regexp: '^JAVA_ARGS='
        line: 'JAVA_ARGS="-Djava.awt.headless=true -Djava.net.preferIPv4Stack=true"'
        
    - name: Add JAVA_HOME to Jenkins default configuration
      lineinfile:
        path: /etc/default/jenkins
        line: 'JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"'
        insertafter: '^JENKINS_ARGS='
        
    - name: Reload systemd daemon
      systemd:
        daemon_reload: yes
      when: jenkins_override.changed
        
    - name: Restart Jenkins service
      systemd:
        name: jenkins
        state: restarted
```

### Exécution du déploiement

```bash
# Vérification de la syntaxe
ansible-playbook site.yml --syntax-check

# Exécution en mode simulation
ansible-playbook site.yml --check

# Déploiement réel
ansible-playbook site.yml

# Si Jenkins ne démarre pas correctement avec Java
ansible-playbook fix_jenkins_java_path.yml
```

## 12. Dépannage

### Problèmes courants et solutions

#### Erreurs de connexion SSH

```bash
# Vérifier la connectivité
ansible all -m ping -vvv

# Vérifier les permissions de la clé SSH
chmod 400 ~/.ssh/ma_cle.pem
```

#### Erreurs d'autorisation

```bash
# Ajouter become: yes pour utiliser sudo
ansible-playbook site.yml --become

# Spécifier un utilisateur sudo différent
ansible-playbook site.yml --become --become-user=root
```

#### Déboguer un playbook

```bash
# Exécution avec plus de verbosité
ansible-playbook site.yml -vvv

# Exécution étape par étape
ansible-playbook site.yml --step
```

## 13. Bonnes pratiques

1. **Organisation du code**
   - Utilisez des rôles pour organiser votre code
   - Gardez les playbooks simples et lisibles
   - Utilisez des noms descriptifs pour les tâches

2. **Gestion des variables**
   - Définissez des valeurs par défaut pour toutes les variables
   - Utilisez group_vars et host_vars pour organiser les variables
   - Documentez toutes les variables

3. **Idempotence**
   - Assurez-vous que vos playbooks peuvent être exécutés plusieurs fois sans effet secondaire
   - Utilisez des modules qui supportent l'idempotence (apt, yum, service, etc.)
   - Évitez les commandes shell/command quand possible

4. **Sécurité**
   - Ne stockez jamais de secrets en clair dans les playbooks
   - Utilisez Ansible Vault pour les données sensibles
   - Limitez les privilèges sudo au minimum nécessaire

5. **Tests**
   - Testez vos playbooks en mode check avant de les exécuter
   - Utilisez des environnements de test avant la production
   - Automatisez les tests avec Molecule ou des outils similaires

## 14. Ressources supplémentaires

- [Documentation officielle d'Ansible](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/) - Dépôt de rôles communautaires
- [Ansible for DevOps](https://www.ansiblefordevops.com/) - Livre de Jeff Geerling
- [Awesome Ansible](https://github.com/ansible-community/awesome-ansible) - Liste de ressources Ansible

---

Ce tutoriel vous a fourni les bases pour commencer avec Ansible. N'hésitez pas à explorer davantage la documentation officielle et à expérimenter avec vos propres playbooks et rôles.
