pipeline {
    agent any

    environment {
        GIT_REPO = "https://github.com/BadamTeja/netflix-clone-react-typescript.git"
        DOCKER_IMAGE = "badamteja/netflix-clone"
        SONARQUBE_ENV = "sonar-server"

        // Nexus
        NEXUS_URL = "http://3.6.87.154:8081/repository/devops-artifacts/"
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
        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    tar -czf dist-${BUILD_NUMBER}.tar.gz dist/

                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file dist-${BUILD_NUMBER}.tar.gz \
                    ${NEXUS_URL}dist-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        // ✅ FIXED (API KEY support)
        stage('Build Docker Image') {
            steps {
                withCredentials([string(
                    credentialsId: 'tmdb-api-key',
                    variable: 'TMDB_KEY'
                )]) {
                    script {
                        docker.build(
                            "${DOCKER_IMAGE}:${BUILD_NUMBER}",
                            "--build-arg TMDB_V3_API_KEY=${TMDB_KEY} ."
                        )
                    }
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
