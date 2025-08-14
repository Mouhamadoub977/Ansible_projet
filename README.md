# Projet Ansible - Déploiement Apache2 et Jenkins

Ce projet Ansible automatise le déploiement d'Apache2 sur deux instances EC2 AWS et l'installation de Jenkins sur l'une d'entre elles.

## Infrastructure

- **Host1**: 13.60.162.182 (ec2-13-60-162-182.eu-north-1.compute.amazonaws.com)
  - Apache2 avec page HTML personnalisée
  - Jenkins (port 8080)
- **Host2**: 13.60.184.126 (ec2-13-60-184-126.eu-north-1.compute.amazonaws.com)
  - Apache2 avec page HTML personnalisée

## Structure du projet

```
ansible/
├── ansible.cfg           # Configuration Ansible
├── inventory             # Inventaire des hôtes
├── site.yml              # Playbook principal
├── roles/
│   ├── apache2/          # Rôle pour Apache2
│   └── jenkins/          # Rôle pour Jenkins
├── fix_jenkins_java_path.yml  # Playbook pour configurer Java 17 pour Jenkins
└── README.md             # Ce fichier
```

## Problèmes rencontrés et solutions

### 1. Configuration SSH

**Problème**: Erreurs de connexion SSH aux instances EC2.

**Solution**: 
- Mise à jour de `ansible.cfg` pour utiliser la clé SSH correcte
- Configuration des permissions de la clé SSH avec `chmod 400 tp_ansible_mouha.pem`
- Utilisation d'un chemin relatif pour la clé SSH pour une meilleure portabilité

### 2. Exécution du rôle Jenkins

**Problème**: Le rôle Jenkins n'était pas exécuté lors du déploiement initial.

**Solution**: Ajout de la section Jenkins manquante dans le playbook principal `site.yml`.

### 3. Échec du démarrage du service Jenkins

**Problème**: Jenkins ne démarrait pas après l'installation, avec l'erreur "Running with Java 11... which is older than the minimum required version (Java 17)".

**Étapes de diagnostic**:
1. Création d'un playbook de diagnostic (`jenkins_debug.yml`) pour vérifier les journaux et l'état du service
2. Identification du problème: Jenkins nécessite Java 17 ou supérieur, mais Java 11 était installé

**Solution**:
1. Installation de Java 17 via un playbook dédié (`fix_jenkins.yml`)
2. Malgré l'installation de Java 17, Jenkins continuait à utiliser Java 11
3. Création d'un playbook (`fix_jenkins_java_path.yml`) pour:
   - Configurer un override systemd pour Jenkins
   - Définir explicitement JAVA_HOME et JENKINS_JAVA_CMD
   - Mettre à jour la configuration dans /etc/default/jenkins
   - Redémarrer le service Jenkins

## Utilisation

### Déploiement complet
```bash
ansible-playbook site.yml
```

### Configuration de Java 17 pour Jenkins (si nécessaire)
```bash
ansible-playbook fix_jenkins_java_path.yml
```

## Accès à Jenkins

- URL: http://13.60.162.182:8080/
- Mot de passe administrateur initial: disponible dans `/var/lib/jenkins/secrets/initialAdminPassword` sur host1

## Bonnes pratiques de sécurité

1. Ne jamais commiter ou partager les clés privées SSH dans le dépôt
2. Utiliser `.gitignore` pour exclure les fichiers sensibles
3. Toujours définir les permissions correctes sur les clés SSH (chmod 400)
4. Partager les instructions pour que les membres de l'équipe utilisent leurs propres copies de clés en toute sécurité
