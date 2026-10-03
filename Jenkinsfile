pipeline {
    agent any

    environment {
        REGISTRY       = 'localhost:5000'
        IMAGE          = 'backend-app:latest'
        FULL_IMAGE     = "${REGISTRY}/backend-app:latest"
        DOCKER_NETWORK = 'devops-net'
        DB_NAME        = 'test_db'
        APP_PORT       = '8089'   // 8080 = Jenkins, 8081 = nginx, 8082 = adminer
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test Maven') {
            steps {
                dir('backend') { sh 'mvn clean package' }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE backend/'
                sh 'docker tag $IMAGE $FULL_IMAGE'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials',
                                 usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | docker login $REGISTRY -u "$REG_USER" --password-stdin
                        docker push $FULL_IMAGE
                    '''
                }
            }
        }

        stage('Déploiement MySQL') {
            steps {
                withCredentials([string(credentialsId: 'mysql-root-password', variable: 'DB_PASS')]) {
                    sh '''
                        docker network inspect $DOCKER_NETWORK >/dev/null 2>&1 || docker network create $DOCKER_NETWORK

                        if [ -z "$(docker ps -q -f name=^mysql$)" ]; then
                            docker rm -f mysql 2>/dev/null || true
                            docker run -d --name mysql \
                              --network $DOCKER_NETWORK \
                              -e MYSQL_ROOT_PASSWORD="$DB_PASS" \
                              -e MYSQL_DATABASE=$DB_NAME \
                              -v mysql-data:/var/lib/mysql \
                              -p 3306:3306 \
                              mysql:8.0
                        fi

                        echo "Attente de MySQL..."
                        for i in $(seq 1 30); do
                            docker exec mysql mysqladmin ping -h localhost -uroot -p"$DB_PASS" --silent && break
                            sleep 2
                        done
                    '''
                }
            }
        }

        stage('Déploiement backend-app') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'docker-registry-credentials',
                                     usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS'),
                    string(credentialsId: 'mysql-root-password', variable: 'DB_PASS')
                ]) {
                    sh '''
                        # 1. Arrêter et supprimer l'ancien conteneur
                        docker stop backend-app 2>/dev/null || true
                        docker rm backend-app 2>/dev/null || true

                        # 2. Récupérer l'image publiée
                        echo "$REG_PASS" | docker login $REGISTRY -u "$REG_USER" --password-stdin
                        docker pull $FULL_IMAGE

                        # 3. Lancer le nouveau conteneur
                        docker run -d --name backend-app \
                          --network $DOCKER_NETWORK \
                          -p $APP_PORT:8080 \
                          -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/$DB_NAME?createDatabaseIfNotExist=true" \
                          -e SPRING_DATASOURCE_USERNAME=root \
                          -e SPRING_DATASOURCE_PASSWORD="$DB_PASS" \
                          $FULL_IMAGE
                    '''
                }
            }
        }

        stage('Vérification') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        docker logs backend-app 2>&1 | grep -q "Started" && break
                        sleep 3
                    done
                    docker ps
                    docker logs backend-app
                    docker logs backend-app 2>&1 | grep -q "Started"
                '''
            }
        }
    }

    post {
        always  { sh 'docker logout localhost:5000 || true' }
        success { echo 'Pipeline CD terminé avec succès' }
        failure { echo 'Échec du pipeline' }
    }
}
