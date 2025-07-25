    pipeline {
    agent any

    environment {
        ACR_LOGIN_SERVER = 'nithish1116.azurecr.io'
        IMAGE_NAME = 'sample-react'
        IMAGE_TAG = "${BUILD_NUMBER}"
        REGISTRY = "${ACR_LOGIN_SERVER}"
        SONAR_TOKEN = credentials('sonarqube-token')  // Jenkins secret text
        SONAR_HOST_URL = 'http://localhost:9000'
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                credentialsId: 'github-credentials',
                url: 'https://github.com/Vehonitor/sample-react-test.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }

        stage('Build React App') {
            steps {
                bat 'npm run build'
            }
        }

        stage('Run SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    bat """
                        echo Running SonarQube scan...
                        npm install -g sonarqube-scanner
                        sonarqube-scanner ^
                          -Dsonar.projectKey=react-app ^
                          -Dsonar.sources=. ^
                          -Dsonar.host.url=%SONAR_HOST_URL% ^
                          -Dsonar.login=%SONAR_TOKEN%
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                bat """
                    echo Running Trivy Scan...
                    trivy image --exit-code 0 --severity CRITICAL,HIGH ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} > trivy-report.txt
                """
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    docker.withRegistry("https://${ACR_LOGIN_SERVER}", 'acr-admin-login2') {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                    bat """
                        powershell -Command "(Get-Content deployment.yaml) -replace 'image: .*', 'image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}' | Set-Content deployment-temp.yaml"
                    """
                    bat "kubectl --kubeconfig=\"%KUBECONFIG%\" apply -f deployment-temp.yaml"
                    bat "kubectl --kubeconfig=\"%KUBECONFIG%\" apply -f service.yaml"
                }
            }
        }
    }
}
