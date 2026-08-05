pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java21'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app"
        RELEASE = "1.0.0"

        DOCKER_USER = "shelly1230897"
        DOCKER_PASS = "dockerhub"

        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/Shelly-Saini/register-app.git'
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        dockerImage = docker.build("${IMAGE_NAME}")
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Install Trivy') {
            steps {
                sh '''
                if ! command -v trivy >/dev/null 2>&1; then
                    echo "Installing Trivy..."

                    sudo apt-get update

                    sudo apt-get install -y wget gnupg lsb-release apt-transport-https

                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
                    sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg

                    echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb stable main" | \
                    sudo tee /etc/apt/sources.list.d/trivy.list

                    sudo apt-get update

                    sudo apt-get install -y trivy
                fi

                trivy --version
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    sh '''
                    trivy image \
                    --no-progress \
                    --scanners vuln \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh '''
                    docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                    docker rmi ${IMAGE_NAME}:latest || true
                    '''
                }
            }
        }
    }
}
