pipeline {
    agent any

    stages {
        
         stage('Pre-check') {
            steps {
                script {
                    // Lấy thông điệp của commit hiện tại
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    echo "Commit Message: ${commitMessage}"
                    
                    // Nếu nhánh không phải main và commit là merge pull request thì dừng pipeline
                    if (env.BRANCH_NAME != 'main' && commitMessage.contains("Merge pull request")) {
                        echo "Nhánh ${env.BRANCH_NAME} là nhánh merge PR. Bỏ qua pipeline."
                        // Dừng pipeline: ta có thể dùng error() để dừng và báo kết quả thành công
                        currentBuild.result = 'SUCCESS'
                        error("Skip build for non-main branch after merge")
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    def branchToCheckout = env.BRANCH_NAME ?: 'main'
                    echo "Checkout branch: ${branchToCheckout}"
                    git branch: branchToCheckout, 
                        url: 'https://github.com/ndmanh3003/spring-petclinic-microservices-1.git'
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    // Get current commit ID for tagging
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    
                    // Define Docker Hub credentials
                    withCredentials([usernamePassword(credentialsId: 'docker_hub_PAT', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Login to Docker Hub
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                        
                        // Get the list of changed files in the current commit
                        def changedFiles = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim().split('\n')
                        
                        // Define the services and their directories
                        def serviceDirs = [
                            'customers-service': 'spring-petclinic-customers-service',
                            'genai-service': 'spring-petclinic-genai-service',
                            'vets-service': 'spring-petclinic-vets-service',
                            'visits-service': 'spring-petclinic-visits-service'
                        ]
                        
                        // Determine which services have changes
                        def servicesToBuild = []
                        for (def file in changedFiles) {
                            for (def entry in serviceDirs) {
                                if (file.startsWith(entry.value + '/')) {
                                    if (!servicesToBuild.contains(entry.key)) {
                                        servicesToBuild.add(entry.key)
                                    }
                                }
                            }
                        }
                        
                        // If no specific service changes detected, skip this stage
                        if (servicesToBuild.isEmpty()) {
                            echo "No specific service changes detected. Skipping Docker image build."
                            return
                        }
                        
                        // Build and push images for services with changes
                        for (def service in servicesToBuild) {
                            def serviceDir = serviceDirs[service]
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${service}"
                            
                            // Build the service JAR file
                            sh "./mvnw -pl ${serviceDir} -am clean package -DskipTests"
                            
                            // Build and push Docker image
                            sh """
                            cp ${serviceDir}/target/*.jar docker/${service}.jar
                            cd docker
                            docker build --build-arg ARTIFACT_NAME=${service} --build-arg EXPOSED_PORT=8080 -t ${imageName}:${commitId} .

                            docker push ${imageName}:${commitId}

                            rm ${service}.jar
                            cd ..
                            """
                            // Only tag/push latest if on main branch
                            if (env.BRANCH_NAME == 'main') {
                                sh """
                                docker tag ${imageName}:${commitId} ${imageName}:latest
                                docker push ${imageName}:latest
                                """
                            }
                        }
                    }
                }
            }
        }

    }

    post {
        success {
            script {
                def commitId = env.GIT_COMMIT
                echo "Sending 'success' status to GitHub for commit: ${commitId}"
                def response = httpRequest(
                    url: "https://api.github.com/repos/ndmanh3003/spring-petclinic-microservices-1/statuses/${commitId}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: """{
                        "state": "success",
                        "description": "Build passed",
                        "context": "ci/jenkins-pipeline",
                        "target_url": "${env.BUILD_URL}"
                    }""",
                    authentication: 'github-token-fix'
                )
                echo "GitHub Response: ${response.status}"
            }
        }

        failure {
            script {
                def commitId = env.GIT_COMMIT
                echo "Sending 'failure' status to GitHub for commit: ${commitId}"
                def response = httpRequest(
                    url: "https://api.github.com/repos/ndmanh3003/spring-petclinic-microservices-1/statuses/${commitId}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: """{
                        "state": "failure",
                        "description": "Build failed",
                        "context": "ci/jenkins-pipeline",
                        "target_url": "${env.BUILD_URL}"
                    }""",
                    authentication: 'github-token-fix'
                )
                echo "GitHub Response: ${response.status}"
            }
        }

        always {
            echo "Pipeline finished."
        }
    }
}
