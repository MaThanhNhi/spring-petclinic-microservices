pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'JDK11'
    }

    environment {
        SERVICES = "api-gateway customers-service vets-service visits-service config-server discovery-server admin-server"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", 
                    url: 'https://github.com/MaThanhNhi/spring-petclinic-microservices.git', 
                    credentialsId: 'github-token'
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = sh(script: "git diff --name-only origin/main...origin/${env.BRANCH_NAME}", returnStdout: true).trim().split('\n')
                    def affectedServices = []

                    for (service in env.SERVICES.split(' ')) {
                        if (changedFiles.any { it.startsWith(service + "/") }) {
                            affectedServices.add(service)
                        }
                    }

                    if (affectedServices.isEmpty()) {
                        echo "No service changed, skipping pipeline."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    env.AFFECTED_SERVICES = affectedServices.join(' ')
                    echo "Affected Services: ${env.AFFECTED_SERVICES}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    for (service in env.AFFECTED_SERVICES.split(' ')) {
                        dir(service) {
                            sh 'mvn test'
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        for (service in env.AFFECTED_SERVICES.split(' ')) {
                            junit "${service}/target/surefire-reports/*.xml"
                            jacoco execPattern: "${service}/target/jacoco.exec"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    for (service in env.AFFECTED_SERVICES.split(' ')) {
                        dir(service) {
                            sh 'mvn clean package'
                        }
                    }
                }
            }
            post {
                success {
                    script {
                        for (service in env.AFFECTED_SERVICES.split(' ')) {
                            archiveArtifacts artifacts: "${service}/target/*.jar", fingerprint: true
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            echo 'Build failed!'
        }
        success {
            echo 'Build succeeded!'
        }
    }
}
