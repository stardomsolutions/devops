pipeline {
   agent any
   environment {
     DOCKER_REGISTRY = "registry.devops:5000"
     VAULT_ADDR = "http://vault.devops:8200"
     VAULT_PATH_MYSQL="kv/mysql/db"
     VAULT_TOKEN_MYSQL="s.E6WxeDlyHOkJcDDrgE0E39fu"
     MYSQL_STAGING_URL="staging.devops:3306"
     MYSQL_PROD_URL="production.devops:3306"
     MYSQL_DB_NAME="test"
     MYSQL_DB_PASSWORD="test"
     MYSQL_DB_USER="test"
     MYSQL_DB_ROOT="tooor"
   }
   stages {
      stage('Build') {
         steps {
            sh '''
               docker images -f dangling=true -q | xargs docker rmi || true
               mvn -f backend/pom.xml clean package
               npm --prefix frontend install
            '''   
         }
      }
      stage('Archive') {
         steps {
            parallel(
               ui: { 
                  sh '''
                        tar czf ${GIT_COMMIT}.tar.gz frontend/
                  '''            
                  script {
                     def remote = [:]
                     remote.name = 'archiver'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'archiver.devops'
                     remote.identityFile = '~/.ssh/archiver.key'
                     sshPut remote: remote, filterRegex: '.tar.gz$',from: '.' ,into: '/home/vagrant/archiver/frontend'
                  }
               },
               api:{
                  sh '''
                        mv ${WORKSPACE}/backend/target/*.jar ${WORKSPACE}/backend/target/${GIT_COMMIT}.jar
                  '''            
                  script {
                     def remote = [:]
                     remote.name = 'archiver'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'archiver.devops'
                     remote.identityFile = '~/.ssh/archiver.key'
                     sshPut remote: remote, filterRegex: '.jar$',from: './backend/target' ,into: '/home/vagrant/archiver/backend'
                  }
               }       
            )
         }              
      }
      stage('Staging Setup') {
         steps {
            parallel(
               ui: { // Prepare the Docker image for the staging ui
                  sh '''
                     docker tag "devops/ui:staging" "${DOCKER_REGISTRY}/devops/ui:staging"
                     docker push "${DOCKER_REGISTRY}/devops/ui:staging"
                     docker rmi "${DOCKER_REGISTRY}/devops/ui:staging"
                  '''
               },
               api: {
                  sh '''
                     docker build --build-arg FILE_NAME=${GIT_COMMIT} -t "devops/api:staging" -f backend/Dockerfile .
                     docker tag "devops/api:staging" "${DOCKER_REGISTRY}/devops/api:staging"
                     docker push "${DOCKER_REGISTRY}/devops/api:staging"
                     docker rmi "${DOCKER_REGISTRY}/devops/api:staging"
                  '''
               },
               db: { // Parallely start the MySQL Daemon in the staging server first stop if already running then start
                  script {
                     def remote = [:]
                     remote.name = 'staging'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'staging.devops'
                     remote.identityFile = '~/.ssh/staging.key'
                     sshCommand remote: remote, command: "docker stop mysqldb backend frontend || true"
                     sshCommand remote: remote, command: "docker rm backend mysqldb frontend || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/api:staging ${DOCKER_REGISTRY}/devops/ui:staging || true"
                     sshCommand remote: remote, command: "docker run -d -p 3306:3306 \
                        -e MYSQL_DATABASE=${MYSQL_DB_NAME} -e MYSQL_ROOT_PASSWORD=${MYSQL_DB_ROOT} -e MYSQL_USER=${MYSQL_DB_USER} -e MYSQL_PASSWORD=${MYSQL_DB_PASSWORD} \
                        -v /home/vagrant/mysql:/var/lib/mysql \
                        --name mysqldb mysql \
                        --default-authentication-plugin=mysql_native_password"
                  }               
               }
            )
         }
      }
      stage('Staging Deploy') {//providing delay for mysql to start
         steps {   
            sh '''
		            sleep 5
            '''
            script {
                def remote = [:]
                remote.name = 'staging'
                remote.user = 'vagrant'
                remote.allowAnyHosts = true
                remote.host = 'staging.devops'
                remote.identityFile = '~/.ssh/staging.key'
                sshCommand remote: remote, command: "docker run -d -p 8080:8080 --link mysqldb -e MYSQL_DB_USER=${MYSQL_DB_USER} \
                  -e MYSQL_DB_PASSWORD=${MYSQL_DB_PASSWORD} -e MYSQL_JDBC_URL=${MYSQL_STAGING_URL} -e MYSQL_DB_NAME=${MYSQL_DB_NAME} \
                  -v /home/vagrant/logs:/home/boot/logs/ --name backend ${DOCKER_REGISTRY}/devops/api:staging"
                sshCommand remote: remote, command: "docker run -d -p 80:80 --link backend \
                  --name frontend ${DOCKER_REGISTRY}/devops/ui:staging"                  
            }
         }
      }
      stage('UAT Tests') {
         steps {   
               echo 'UAT Tests'
         }
      }
      stage('Production Setup') {
         steps {
            parallel(
               ui: { // Prepare the Docker image for the staging ui
                  sh '''
                     docker tag "devops/ui:prod" "${DOCKER_REGISTRY}/devops/ui:prod"
                     docker push "${DOCKER_REGISTRY}/devops/ui:prod"
                     docker rmi "${DOCKER_REGISTRY}/devops/ui:prod"
                  '''
               },
               api: {
                  sh '''
                     docker build --build-arg FILE_NAME=${GIT_COMMIT} -t "devops/api:prod" -f backend/Dockerfile .
                     docker tag "devops/api:prod" "${DOCKER_REGISTRY}/devops/api:prod"
                     docker push "${DOCKER_REGISTRY}/devops/api:prod"
                     docker rmi "${DOCKER_REGISTRY}/devops/api:prod"
                  '''
               },
               db: { 
                  script {
                     def remote = [:]
                     remote.name = 'staging'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'staging.devops'
                     remote.identityFile = '~/.ssh/staging.key'
                     sshCommand remote: remote, command: "docker stop mysqldb backend frontend || true"
                     sshCommand remote: remote, command: "docker rm backend mysqldb frontend || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/api:production ${DOCKER_REGISTRY}/devops/ui:production || true"
                     sshCommand remote: remote, command: "docker run -d -p 3306:3306 \
                        -e MYSQL_DATABASE=${MYSQL_DB_NAME} -e MYSQL_ROOT_PASSWORD=${MYSQL_DB_ROOT} -e MYSQL_USER=${MYSQL_DB_USER} -e MYSQL_PASSWORD=${MYSQL_DB_PASSWORD} \
                        -v /home/vagrant/mysql:/var/lib/mysql \
                        --name mysqldb mysql \
                        --default-authentication-plugin=mysql_native_password"
                  }               
               }
            )
         }
      }
      stage('Production Deploy Approval') {
         steps {
            script {
                  input message: 'Do you approve Deployment ?', ok: 'OK'
            }
         }
      }      
      stage('Production Deploy') {       
         steps {   
            sh '''
		            sleep 5
            '''
            script {
                def remote = [:]
                remote.name = 'production'
                remote.user = 'vagrant'
                remote.allowAnyHosts = true
                remote.host = 'production.devops'
                remote.identityFile = '~/.ssh/production.key'
                sshCommand remote: remote, command: "docker run -d -p 8080:8080 --link mysqldb -e MYSQL_DB_USER=${MYSQL_DB_USER} \
                  -e MYSQL_DB_PASSWORD=${MYSQL_DB_PASSWORD} -e MYSQL_JDBC_URL=${MYSQL_STAGING_URL} -e MYSQL_DB_NAME=${MYSQL_DB_NAME} \
                  -v /home/vagrant/logs:/home/boot/logs/ --name backend ${DOCKER_REGISTRY}/devops/api:production"
                sshCommand remote: remote, command: "docker run -d -p 80:80 --link backend \
                  --name frontend ${DOCKER_REGISTRY}/devops/ui:production"                  
            }
         }
      }                       
   }
}
