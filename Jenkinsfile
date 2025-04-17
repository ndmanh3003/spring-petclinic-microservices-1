pipeline {
    agent any

    stages {
        
        // Stage for pre-checking the commit message and skipping build for merge PR
        stage('Pre-check') {
            steps {
                script {
                    // Retrieve the latest commit message
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    echo "Commit Message: ${commitMessage}"

                    // Skip build if not on the 'main' branch and the commit is a merge from a pull request
                    if (env.BRANCH_NAME != 'main' && commitMessage.contains("Merge pull request")) {
                        echo "Skipping pipeline for PR merge on branch: ${env.BRANCH_NAME}"
                        currentBuild.result = 'SUCCESS'
                        error("Skip build for non-main branch after merge")
                    }
                }
            }
        }

        // Checkout the appropriate branch of the repository
        stage('Checkout Code') {
            steps {
                script {
                    def branchToCheckout = env.BRANCH_NAME ?: 'main'
                    echo "Checking out branch: ${branchToCheckout}"
                    git branch: branchToCheckout, url: 'https://github.com/ndmanh3003/spring-petclinic-microservices-1.git'
                }
            }
        }

        // Build Docker images for changed services
        stage('Build Docker Images') {
            steps {
                script {
                    // Get the current commit ID for tagging Docker images
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    // Use credentials for Docker login
                    withCredentials([usernamePassword(credentialsId: 'docker_hub_PAT', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Docker login
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"

                        // Get the list of changed files in the current commit
                        def changedFiles = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim().split('\n')

                        // Define directories for each service
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

                        // Skip if no services are changed
                        if (servicesToBuild.isEmpty()) {
                            echo "No changes detected for specific services. Skipping Docker build."
                            return
                        }

                        // Build and push Docker images for the changed services
                        for (def service in servicesToBuild) {
                            def serviceDir = serviceDirs[service]
                            def imageName = "${DOCKER_USERNAME}/spring-petclinic-${service}"

                            // Build the JAR for the service
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
                            // If on the main branch, tag and push the latest version
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
                // Notify GitHub of the successful build
                def commitId = env.GIT_COMMIT
                echo "Sending success status to GitHub for commit: ${commitId}"
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
                // Notify GitHub of the failed build
                def commitId = env.GIT_COMMIT
                echo "Sending failure status to GitHub for commit: ${commitId}"
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
