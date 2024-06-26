pipeline {
    agent any 

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DEPLOY_CREDENTIALS = credentials('deploy-credentials')
        DOCKER_USERNAME = 'mecit35'
        DEPLOY_MACHINE = '192.168.1.124'
        CONTAINER_NAME = 'my-container'
    }

    stages {
        stage('Clone repository') {
            steps {
                sh 'rm -rf simple-java-container-CI-CD || true'
                sh 'git clone https://github.com/Mecit-tuksoy/simple-java-container-CI-CD.git'
            }
        }

        stage('Package Application') {
            steps {
                echo 'Compiling source code'
                sh '. ./jenkins/package-application.sh'
            }
        }
        
        stage('Prepare Tags for Docker Images') {
            steps {
                echo 'Preparing Tags for Docker Images'
                script {
                    MVN_VERSION = sh(script: '. ${WORKSPACE}/target/maven-archiver/pom.properties && echo $version', returnStdout: true).trim()
                    env.IMAGE_TAG = "my-java-app-v${MVN_VERSION}".toLowerCase()
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR')]) {
                        sh 'docker stop $(docker ps -q --filter "ancestor=${DOCKER_USERNAME}/${IMAGE_TAG}") || true'
                        sh 'docker rm $(docker ps -a -q --filter "ancestor=${DOCKER_USERNAME}/${IMAGE_TAG}") || true'
                        sh 'docker rmi ${DOCKER_USERNAME}/${IMAGE_TAG} || true'
                        sh 'docker build --force-rm -t ${DOCKER_USERNAME}/${IMAGE_TAG} .'
                    }
                }
            }            
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR')]) {
                        sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                        sh "docker tag ${DOCKER_USERNAME}/${IMAGE_TAG}:latest ${DOCKER_USERNAME}/${IMAGE_TAG}:latest"
                        sh "docker push ${DOCKER_USERNAME}/${IMAGE_TAG}:latest"
                        }
                    }
                }
            }
        
    

        stage('Deploy on other linux machine') {
            steps {
                // Combine both credentials and use them in a single sh block
                withCredentials([
                    usernamePassword(credentialsId: 'deploy-credentials', passwordVariable: 'DEPLOY_PASSWORD', usernameVariable: 'DEPLOY_USER'),
                    usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR')]) {
                    sh '''
                      sshpass -p "${DEPLOY_PASSWORD}" ssh -t -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_MACHINE} "
                        echo \"$DOCKERHUB_CREDENTIALS_PSW\" | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin;
                        docker stop ${CONTAINER_NAME} || true;
                        docker rm ${CONTAINER_NAME} || true;
                        docker rmi ${DOCKERHUB_CREDENTIALS_USR}/${IMAGE_TAG} || true;
                        docker pull ${DOCKERHUB_CREDENTIALS_USR}/${IMAGE_TAG}:latest;
                        docker run -d --name ${CONTAINER_NAME} -p 8080:8080 ${DOCKERHUB_CREDENTIALS_USR}/${IMAGE_TAG}:latest;
                        sleep 30;
                        curl http://${DEPLOY_MACHINE}:8080
                      " 
                    '''
                }
            }
        }
    }        
}

