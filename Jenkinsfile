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
        K8S_GIT_REPO = "https://github.com/nhan925/devops_lab02_k8s.git"
        KUBECONFIG = credentials('kubectl_config')
        SERVICES = "spring-petclinic-admin-server,spring-petclinic-api-gateway,spring-petclinic-config-server,spring-petclinic-discovery-server,spring-petclinic-customers-service,spring-petclinic-vets-service,spring-petclinic-visits-service"
    }
    
    stages {
        stage('Detect Changes') {
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
        
        stage('Build & Test') {
            when { expression { return !CHANGED_SERVICES.isEmpty() } }
            steps {
                script {
                    def parallelStages = [:] // Initialize an empty map for parallel stages
                
                    // Split CHANGED_SERVICES and create a parallel stage for each service
                    CHANGED_SERVICES.split(',').each { service ->
                        parallelStages["Verify ${service}"] = {
                            sh "./mvnw verify -pl ${service}"
                        }
                    }
                
                    // Run the parallel stages
                    parallel parallelStages
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    
                    // Make the build unstable if coverage is below threshold
                    recordCoverage(
                        tools: [[parser: 'JACOCO']],
                        sourceDirectories: [[path: 'src/main/java']],
                        sourceCodeRetention: 'EVERY_BUILD',
                        qualityGates: [
                            [threshold: env.MINIMUM_COVERAGE.toInteger(), metric: 'LINE', baseline: 'PROJECT', unstable: true]
                        ]
                    )
                    
                    // Now check if build became unstable due to coverage, and fail it explicitly
                    // script {
                    //     if (currentBuild.result == 'UNSTABLE') {
                    //         error "Build failed: Line is below the required minimum ${env.MINIMUM_COVERAGE}%"
                    //     }
                    // }
                }
            }
        }

        stage('Login Docker') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    """
                }
            }
        }

        stage('Build Docker Images') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                script {
                    sh "whoami"

                    if (env.GIT_TAG) {
                        CONTAINER_TAG = "${env.GIT_TAG}"
                        CHANGED_SERVICES = env.SERVICES
                        echo "Building all services for tag: ${CONTAINER_TAG}"
                    }
                    else if (env.BRANCH_NAME == 'main') {
                        CONTAINER_TAG = "latest"
                    }
                    else {
                        CONTAINER_TAG = "${env.GIT_COMMIT.take(7)}"
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

        stage('Clean Docker Images & Logout') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID } }
            steps {
                script {
                    echo "Cleaning up Docker images"
                    sh "docker image prune -a -f"
                    sh  "docker logout"
                    echo "Docker logout completed"
                }
            }
        }

        stage('Config kubectl') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID && (env.GIT_TAG || env.BRANCH_NAME == 'main') } }
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubectl_config', variable: 'KUBECONFIG')]) {
                        sh """
                            mkdir -p ~/.kube
                            cp ${KUBECONFIG} ~/.kube/config
                            chmod 777 ~/.kube/config
                            echo "Kubeconfig copied and permissions set"
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when { expression { return !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID && (env.GIT_TAG || env.BRANCH_NAME == 'main') } }
            steps {
                script {
                    sh """
                        sudo apt install yq -y
                        git clone ${K8S_GIT_REPO} k8s
                        cd k8s
                    """

                    if (env.GIT_TAG) {
                        echo "Deploying to Kubernetes with tag: ${env.GIT_TAG}"
                        sh """
                            yq e '.imageTag = "&tag ${env.GIT_TAG}"' -i environments/values-staging.yaml
                        """
                    } else {
                        echo "Deploying to Kubernetes with branch: ${env.BRANCH_NAME}"
                    }

                    // Update the Chart version
                    sh """
                        old_version=\$(yq e '.version' Chart.yaml)
                        IFS='.' read -r major minor patch <<< "\$old_version"
                        new_patch=\$((patch+1))
                        new_version="\${major}.\${minor}.\${new_patch}"
                        yq e '.version = strenv(new_version)' -i Chart.yaml
                    """

                    // Commit and push changes
                    sh """
                        git add .
                        git commit -m "Update imageTag to ${env.GIT_TAG} and bump chart version to \$new_version"
                        git push origin main
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed with result: ${currentBuild.currentResult}"
            echo "Pipeline run by: ${currentBuild.getBuildCauses()[0].userId ?: 'unknown'}"
            echo "Completed at: ${new java.text.SimpleDateFormat('yyyy-MM-dd HH:mm:ss').format(new Date())}"
            cleanWs()

            if (currentBuild.result != 'FAILED' && !CHANGED_SERVICES.isEmpty() && !env.CHANGE_ID && (env.GIT_TAG || env.BRANCH_NAME == 'main')) {
                if (env.GIT_TAG) {
                    echo "Deployment to Kubernetes with tag ${env.GIT_TAG} was successful."
                    echo "Add this to your /etc/hosts file: 172.28.81.156:32211 staging.pet-clinic.cloud"
                } else {
                    echo "Deployment to Kubernetes with branch main was successful."
                    echo "Add this to your /etc/hosts file: 172.28.81.156:32212 dev.pet-clinic.cloud"
                }
            }
        }
    }
}
