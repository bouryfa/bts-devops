# DevOps Stack - BTS

Stack DevOps complet pour l'intégration continue et l'analyse de qualité de code, comprenant Jenkins, SonarQube et un reverse proxy Nginx.

## 🏗️ Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Nginx Proxy   │────│     Jenkins      │    │   SonarQube     │
│   (Port 80)     │    │   (Port 8080)    │    │   (Port 9000)   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │              ┌─────────────────┐
         │                       │              │   PostgreSQL    │
         │                       │              │   (Port 5432)   │
         │                       │              └─────────────────┘
         │                       │
    ┌─────────────────────────────────────────────────────────────┐
    │                    Docker Networks                          │
    │  • devops-frontend (nginx ↔ services)                     │
    │  • devops-jenkins-network (jenkins interne)               │
    │  • devops-sonarqube-network (sonarqube ↔ postgresql)      │
    └─────────────────────────────────────────────────────────────┘
```

## 📦 Services Inclus

### 🔄 Jenkins CI/CD
- **Image**: `jenkins/jenkins:jdk21`
- **Accès**: `jenkins.bts.io` (via nginx)
- **Fonctionnalités**:
  - Intégration continue et déploiement continu
  - Support Docker-in-Docker pour les builds containerisés
  - Interface web complète pour la gestion des pipelines

### 🔍 SonarQube Code Quality
- **Image**: `sonarqube:10.7.0-community`
- **Accès**: `quality.bts.io` (via nginx)
- **Fonctionnalités**:
  - Analyse statique de code
  - Détection de bugs, vulnérabilités et code smells
  - Métriques de qualité et couverture de code

### 🗄️ PostgreSQL Database
- **Image**: `postgres:16.4`
- **Usage**: Base de données dédiée pour SonarQube
- **Persistance**: Volume Docker pour la sauvegarde des données

### 🌐 Nginx Reverse Proxy
- **Image**: `nginx:latest`
- **Port**: `80`
- **Fonctionnalités**:
  - Routage par nom de domaine
  - Configuration personnalisable via `conf.d/`

## 🚀 Démarrage Rapide

### Prérequis
- Docker Engine 20.10+
- Docker Compose 2.0+
- 4GB RAM minimum
- 10GB espace disque libre

### 1. Configuration des domaines (optionnel)
Ajoutez ces entrées à votre fichier `/etc/hosts` pour l'accès local :
```bash
127.0.0.1 jenkins.bts.io
127.0.0.1 quality.bts.io
```

### 2. Lancement du stack
```bash
# Cloner le projet
cd devops-stack

# Démarrer tous les services
docker-compose up -d

# Vérifier le statut
docker-compose ps
```

### 3. Accès aux services

| Service | URL | Identifiants par défaut |
|---------|-----|------------------------|
| Jenkins | http://jenkins.bts.io ou http://localhost:80 | Voir logs pour le mot de passe initial |
| SonarQube | http://quality.bts.io | admin / admin |

## 🔧 Configuration Initiale

### Jenkins
1. Récupérer le mot de passe initial :
```bash
docker-compose logs jenkins | grep -A 5 "Please use the following password"
```

2. Accéder à Jenkins et suivre l'assistant d'installation
3. Installer les plugins recommandés
4. Créer un utilisateur administrateur

### SonarQube
1. Se connecter avec `admin/admin`
2. Changer le mot de passe par défaut
3. Configurer les projets et les règles de qualité

## 📁 Structure du Projet

```
devops-stack/
├── docker-compose.yaml          # Configuration principale
├── conf.d/                      # Configuration Nginx
│   ├── default.conf            # Configuration par défaut
│   ├── jenkins.conf            # Proxy vers Jenkins
│   ├── sonarqube.conf          # Proxy vers SonarQube
│   └── [autres-services]/      # Configurations futures
└── README.md                   # Cette documentation
```

## 🔨 Commandes Utiles

### Gestion des services
```bash
# Démarrer le stack
docker-compose up -d

# Arrêter le stack
docker-compose down

# Redémarrer un service spécifique
docker-compose restart jenkins

# Voir les logs
docker-compose logs -f jenkins
docker-compose logs -f sonarqube

# Mise à jour des images
docker-compose pull
docker-compose up -d
```

### Maintenance
```bash
# Sauvegarder les volumes
docker run --rm -v devops-jenkins-data:/data -v $(pwd):/backup alpine tar czf /backup/jenkins-backup.tar.gz -C /data .

# Nettoyer les ressources inutilisées
docker system prune -f
docker volume prune -f
```

## 🔒 Sécurité

### Recommandations de Production
1. **Changer les mots de passe par défaut**
   - PostgreSQL : `POSTGRES_PASSWORD` dans docker-compose.yaml
   - SonarQube : mot de passe admin

2. **Utiliser HTTPS**
   - Configurer SSL/TLS dans Nginx
   - Obtenir des certificats Let's Encrypt

3. **Sécuriser l'accès**
   - Configurer l'authentification LDAP/SSO
   - Limiter l'accès réseau avec des firewalls

4. **Sauvegardes régulières**
   - Automatiser la sauvegarde des volumes Docker
   - Tester la restauration périodiquement

## 🐛 Dépannage

### Problèmes courants

**Jenkins ne démarre pas**
```bash
# Vérifier les logs
docker-compose logs jenkins

# Vérifier les permissions du socket Docker
ls -la /var/run/docker.sock
```

**SonarQube erreur de base de données**
```bash
# Vérifier que PostgreSQL est démarré
docker-compose ps devops-sonarqube-database

# Vérifier la connectivité réseau
docker-compose exec sonarqube ping devops-sonarqube-database
```

**Nginx erreur 502**
```bash
# Vérifier que les services backend sont accessibles
docker-compose exec nginx-reverse-proxy nslookup devops-jenkins
docker-compose exec nginx-reverse-proxy nslookup devops-sonarqube
```

### Logs et monitoring
```bash
# Surveiller tous les services
docker-compose logs -f

# Vérifier l'utilisation des ressources
docker stats

# Inspecter les réseaux
docker network ls
docker network inspect devops-frontend
```

## 🔄 Intégration CI/CD

### Pipeline Jenkins basique
```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/votre-repo/projet.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
    }
}
```

## 📈 Évolutions Futures

Le dossier `conf.d/` contient des configurations pour d'autres services :
- Grafana (monitoring)
- Prometheus (métriques)
- Harbor (registry Docker)
- Applications métier (flask-app, moodboard)

Ces services peuvent être ajoutés au stack selon les besoins.

## 📞 Support

Pour toute question ou problème :
1. Consulter les logs : `docker-compose logs [service]`
2. Vérifier la documentation officielle des services
3. Créer une issue dans le projet

---

**Version**: 1.0  
**Dernière mise à jour**: $(date +%Y-%m-%d)