pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials'
        SONARQUBE_ENV = "MySonarQube"
        TRIVY_TOKEN = credentials('trivy-token') // nếu có
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/13octt/todo-microservices-devops.git'
            }
        }

        stage('Build code with Docker Compose') {
            steps {
                script {
                    sh 'docker-compose up --build -d'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh '${SCANNER_HOME}/bin/sonar-scanner --version'
                        sh """
                            ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=todo \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://192.168.1.69:9000 \
                                -Dsonar.token=sqp_04513c2ea71a49de2b8321de03a83d514702c4fd
                        """
                    }
                }
            }
        }

        stage('Security Check with Trivy') {
            steps {
                script {
                    // Quét bảo mật cho tất cả các image được xây dựng
                    def services = ['auth-service', 'gateway', 'profile-service', 'task-service', 'todo-fe']
                    services.each { service ->
                        sh "trivy image --exit-code 1 --severity HIGH yourdockerhubusername/${service}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        // Áp dụng tất cả các tệp YAML trong thư mục k8s
                        sh "kubectl apply -f ./k8s/ --kubeconfig=$KUBECONFIG"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Dọn dẹp container và mạng nếu đang chạy
                sh 'docker-compose down'
            }
        }

        success {
            script {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
