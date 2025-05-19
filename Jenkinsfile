def CHANGED_SERVICES = ""
def IGNORED_DIR = [".github", ".mvn", "docs", "LICENSE", "README.md", "mvnw"]

pipeline {
    agent any
    
    tools {
        jdk 'JDK17'
    }
    
    environment {
        MINIMUM_COVERAGE = 70
        DOCKER_REGISTRY = "nhan925"
        SERVICES = "spring-petclinic-admin-server,spring-petclinic-api-gateway,spring-petclinic-config-server,spring-petclinic-discovery-server,spring-petclinic-customers-service,spring-petclinic-vets-service,spring-petclinic-visits-service,spring-petclinic-genai-service"
    }
    
    stages {
        stage('Detect Release') {
            when { expression { return env.TAG_NAME } }
            steps {
                script {
                    echo "A new release found with tag ${env.TAG_NAME}"
                    CHANGED_SERVICES = env.SERVICES
                }
            }
        }
        
        stage('Detect Changes') {
            when { expression { return !env.TAG_NAME} }
            steps {
                script {                  
                    def compareTarget = env.CHANGE_TARGET ? "origin/${env.CHANGE_TARGET}" : "HEAD~1"
                    def changedFiles = sh(script: "git diff --name-only ${compareTarget}", returnStdout: true).trim()
                    
                    def changedServicesList = []
                    env.SERVICES.split(",").each { service ->
                        if (changedFiles.split("\n").any { it.startsWith(service) }) {
                            changedServicesList.add(service)
                        }
                    }
                    
                    CHANGED_SERVICES = changedServicesList.join(",")
                    
                    if (CHANGED_SERVICES.isEmpty() && 
                        !changedFiles.split("\n").any { file ->
                            IGNORED_DIR.any { dir -> file.startsWith(dir) }
                        }) {
                        CHANGED_SERVICES = env.SERVICES
                    }
                    
                    echo "Services to build: ${CHANGED_SERVICES ?: 'None'}"
                }
            }
        }
        
        // stage('Build & Test') {
        //     when { expression { return !CHANGED_SERVICES.isEmpty() && !env.TAG_NAME } }
        //     steps {
        //         script {
        //             def parallelStages = [:] // Initialize an empty map for parallel stages
                
        //             // Split CHANGED_SERVICES and create a parallel stage for each service
        //             CHANGED_SERVICES.split(',').each { service ->
        //                 parallelStages["Verify ${service}"] = {
        //                     sh "./mvnw verify -pl ${service}"
        //                 }
        //             }
                
        //             // Run the parallel stages
        //             parallel parallelStages
        //         }
        //     }
        //     post {
        //         always {
        //             junit '**/target/surefire-reports/*.xml'
                    
        //             // Make the build unstable if coverage is below threshold
        //             recordCoverage(
        //                 tools: [[parser: 'JACOCO']],
        //                 sourceDirectories: [[path: 'src/main/java']],
        //                 sourceCodeRetention: 'EVERY_BUILD',
        //                 qualityGates: [
        //                     [threshold: env.MINIMUM_COVERAGE.toInteger(), metric: 'LINE', baseline: 'PROJECT', unstable: true]
        //                 ]
        //             )
                    
        //             // Now check if build became unstable due to coverage, and fail it explicitly
        //             // script {
        //             //     if (currentBuild.result == 'UNSTABLE') {
        //             //         error "Build failed: Line is below the required minimum ${env.MINIMUM_COVERAGE}%"
        //             //     }
        //             // }
        //         }
        //     }
        // }

        stage('Login Docker') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Build Docker Images') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                script {
                    sh "whoami"

                    if (env.TAG_NAME) {
                        CONTAINER_TAG = env.TAG_NAME
                        echo "Building all services for tag: ${CONTAINER_TAG}"
                    }
                    else {
                        CONTAINER_TAG = "${env.GIT_COMMIT.take(7)}"
                        echo "Building all services for commit: ${CONTAINER_TAG}"
                    }

                    def parallelStages = [:] // Initialize an empty map for parallel stages
                    
                    // Split CHANGED_SERVICES and create a parallel stage for each service
                    CHANGED_SERVICES.split(',').each { service ->
                        parallelStages["Building Docker image for ${service}"] = {
                            sh "./mvnw clean install -pl ${service} -Dmaven.test.skip=true -P buildDocker -Ddocker.image.prefix=${env.DOCKER_REGISTRY} -Ddocker.image.tag=${CONTAINER_TAG} -Dcontainer.build.extraarg=\"--push\""
                        }
                    }
                    
                    // Run the parallel stages
                    parallel parallelStages
                }
            }
        }

        // may make conflict when >1 jobs running on the same agent
        stage('Clean Docker Images & Logout') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                script {
                    echo "Cleaning up Docker images"
                    sh "docker system prune -af"
                    sh  "docker logout"
                    echo "Docker logout completed"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID && (env.TAG_NAME || env.BRANCH_NAME == 'main') } }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh ''' 
                            git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/nhan925/devops_lab02_k8s.git k8s
                            cd k8s

                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"
                        '''
                    }

                    sh '''
                        cd k8s

                        # Extract old version using grep + cut
                        old_version=$(grep '^version:' Chart.yaml | cut -d' ' -f2)
                        echo "Old version: $old_version"

                        major=$(echo "$old_version" | cut -d. -f1)
                        minor=$(echo "$old_version" | cut -d. -f2)
                        patch=$(echo "$old_version" | cut -d. -f3)
                        
                        new_patch=$((patch + 1))
                        new_version="$major.$minor.$new_patch"
                        echo "New version: $new_version"

                        # Update version using sed
                        sed -i "s/^version: .*/version: $new_version/" Chart.yaml
                    '''

                    if (env.TAG_NAME) {
                        echo "Deploying to Kubernetes with tag: ${env.TAG_NAME}"
                        COMMIT_MESSAGE = "Deploy for tag ${env.TAG_NAME}"
                        sh '''
                            cd k8s
                            sed -i "s/^imageTag: .*/imageTag: \\&tag ${TAG_NAME}/" environments/values-staging.yaml
                        ''' 
                        echo "✅ Updated tag for all services to ${env.TAG_NAME}"
                    } else {
                        echo "Deploying to Kubernetes with branch: main"
                        CHANGED_SERVICES.split(',').each { fullName ->
                            def shortName = fullName.replaceFirst('spring-petclinic-', '')
                            def shortCommit = env.GIT_COMMIT.take(7)

                            sh """
                                cd k8s
                                sed -i '/${shortName}:/{n;n;s/tag:.*/tag: ${shortCommit}/}' environments/values-dev.yaml
                            """
                            echo "✅ Updated tag for ${shortName} to ${env.GIT_COMMIT.take(7)}"
                        }
                        
                        COMMIT_MESSAGE = "Deploy for branch main with commit ${env.GIT_COMMIT.take(7)}"
                    }

                    sh """
                        cd k8s
                        git add .
                        git commit -m "${COMMIT_MESSAGE}"
                        git push origin main
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline completed with result: ${currentBuild.currentResult}"
                echo "Pipeline run by: ${currentBuild.getBuildCauses()[0].userId ?: 'unknown'}"
                echo "Completed at: ${new java.text.SimpleDateFormat('yyyy-MM-dd HH:mm:ss').format(new Date())}"
                cleanWs()

                if (currentBuild.result != 'FAILED' && !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID && (env.TAG_NAME || env.BRANCH_NAME == 'main')) {
                    def baseDomain = "petclinic.cloud"
                    def envPrefix = env.TAG_NAME ? "staging" : "dev"
                    def nodeIP = "172.28.81.63" // change to match the worker node's IP

                    echo "✅ Deployment to Kubernetes was successful with environment: ${envPrefix}"
                    echo "Add this to your /etc/hosts file:"
                    echo "${nodeIP} ${envPrefix}-${baseDomain}"
                    echo "${nodeIP} eureka.${envPrefix}-${baseDomain}"

                    echo "Access your app:"
                    echo "🔗 HTTP:  http://${envPrefix}-${baseDomain}:32080"
                    echo "🔒 HTTPS: https://${envPrefix}-${baseDomain}:32443"
                }
            }
        }
    }
}
