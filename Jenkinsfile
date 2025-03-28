pipeline {
    agent any  // Sử dụng bất kỳ agent nào có sẵn

    tools {
        maven 'Maven3'   // Cấu hình Maven từ Jenkins Global Tool Configuration
        jdk 'JDK11'      // Cấu hình JDK
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/MaThanhNhi/spring-petclinic-microservices.git', credential: 'github-token'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'  // Chạy unit test
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'  // Upload test result
                    jacoco execPattern: '**/target/jacoco.exec'  // Upload test coverage (nếu dùng Jacoco)
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'  // Build project
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
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
