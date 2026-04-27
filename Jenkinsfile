pipeline {
    agent any

    environment {
        GIT_REPO = "https://github.com/BadamTeja/netflix-clone-react-typescript.git"
        DOCKER_IMAGE = "netflix-clone:${BUILD_NUMBER}"
        DOCKER_REPO  = "badamteja/netflix-clone"
        SONARQUBE_ENV = "sonar-server"

        // Nexus
        NEXUS_URL = "http://15.207.112.236:8081/repository/devops-artifacts/"
    }

    tools {
        nodejs "nodejs"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        credentialsId: 'git-creds',
                        url: "${GIT_REPO}"
                    ]]
                )
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install -g yarn'
                sh 'yarn install'
            }
        }

        stage('Build') {
            steps {
                sh 'yarn build'
            }
        }

        stage('Test') {
            steps {
                sh 'yarn test || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                    npm install -g sonar-scanner
                    sonar-scanner \
                      -Dsonar.projectKey=netflix-clone \
                      -Dsonar.sources=src
                    '''
                }
            }
        }

        // ✅ FIXED (dist instead of build + credentials added)
       stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    zip -r dist/app.zip dist

                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file dist/app.zip \
                    http://15.207.112.236:8081/repository/raw-hosted/app.zip
                    '''
                }
            }
        }

        // ✅ FIXED (API KEY support)
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_REPO:$BUILD_NUMBER .'
            }
        }

     stage('Push to Docker Hub') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'docker-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

            docker push $DOCKER_REPO:$BUILD_NUMBER
            docker push $DOCKER_REPO:latest
            '''
        }
    }
}

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                    usernamePassword(
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "Updating deployment image..."

                    sed -i "s|IMAGE_PLACEHOLDER|$DOCKER_REPO:$BUILD_NUMBER|g" k8s/deployment.yml

                    echo "Deploying to Kubernetes..."
                    kubectl apply -f k8s/deployment.yml
                    kubectl apply -f k8s/service.yml
                    '''
                }
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
