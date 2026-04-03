pipeline {
    agent any

    environment {
        GIT_REPO = "https://github.com/BadamTeja/netflix-clone-react-typescript.git"
        DOCKER_IMAGE = "badamteja/netflix-clone"
        SONARQUBE_ENV = "sonar-scanner"

        // Nexus
        NEXUS_URL = "http://3.6.87.154/:8081/repository/devops-artifacts/"
        NEXUS_CREDENTIALS = "nexus-creds"
    }

    tools {
        nodejs "nodejs"
    }

    stages {

        stage('Checkout Code') {
            steps {
              checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'git-creds', url: 'https://github.com/BadamTeja/netflix-clone-react-typescript.git']])
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test -- --watchAll=false || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                    npm install -g sonar-scanner
                    sonar-scanner \
                      -Dsonar.projectKey=netflix-clone \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=$SONAR_HOST_URL \
                      -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        // 🔥 THIS IS WHERE NEXUS COMES (NO pom.xml needed)
        stage('Upload Artifact to Nexus') {
            steps {
                script {
                    sh '''
                    tar -czf build-${BUILD_NUMBER}.tar.gz build/

                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file build-${BUILD_NUMBER}.tar.gz \
                    ${NEXUS_URL}build-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('', 'docker-creds') {
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                sed -i 's|IMAGE_TAG|${BUILD_NUMBER}|g' k8s/deployment.yaml
                kubectl apply -f k8s/
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline Success"
        }
        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
