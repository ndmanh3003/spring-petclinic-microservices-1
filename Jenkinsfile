pipeline {
    agent any

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    echo "🔍 Kiểm tra xem kho lưu trữ có phải là shallow không..."
                    // Kiểm tra xem kho lưu trữ có phải là shallow repository không
                    def isShallow = sh(script: "git rev-parse --is-shallow-repository", returnStdout: true).trim()
                    echo "⏳ Kho lưu trữ có phải shallow không? ${isShallow}"

                    // Nếu là shallow, thực hiện fetch toàn bộ lịch sử commit
                    if (isShallow == "true") {
                        echo "📂 Kho lưu trữ là shallow. Đang lấy toàn bộ lịch sử..."
                        sh 'git fetch origin main --prune --unshallow'
                    } else {
                        echo "✅ Kho lưu trữ đã đầy đủ. Bỏ qua bước --unshallow."
                        sh 'git fetch origin main --prune'
                    }

                    // Fetch tất cả các nhánh để đảm bảo có dữ liệu mới nhất
                    echo "📂 Đang fetch tất cả các nhánh..."
                    sh 'git fetch --all --prune'

                    // Kiểm tra xem nhánh main có tồn tại không
                    echo "🔍 Kiểm tra xem nhánh origin/main có tồn tại không..."
                    def mainExists = sh(script: "git branch -r | grep 'origin/main' || echo ''", returnStdout: true).trim()

                    // Nếu không tồn tại, thêm nhánh origin/main và fetch lại
                    if (!mainExists) {
                        echo "❌ origin/main không tồn tại trong remote. Đang fetch tất cả nhánh..."
                        sh 'git remote set-branches --add origin main'
                        sh 'git fetch --all'

                        mainExists = sh(script: "git branch -r | grep 'origin/main' || echo ''", returnStdout: true).trim()

                        // Nếu sau khi fetch vẫn không có nhánh main, báo lỗi
                        if (!mainExists) {
                            error("❌ origin/main vẫn không tồn tại! Hãy kiểm tra nhánh này trên remote.")
                        }
                    }

                    // Lấy commit base để so sánh với commit hiện tại
                    def baseCommit = sh(script: "git merge-base origin/main HEAD", returnStdout: true).trim()
                    echo "🔍 Commit base: ${baseCommit}"

                    // Kiểm tra commit base có hợp lệ không
                    if (!baseCommit) {
                        error("❌ Không tìm thấy commit base!")
                    }

                    // Lấy danh sách các file thay đổi từ commit base
                    def changes = sh(script: "git diff --name-only ${baseCommit} HEAD", returnStdout: true).trim()

                    echo "📜 Các file thay đổi:\n${changes}"

                    // Nếu không có thay đổi nào, bỏ qua bước build và test
                    if (!changes) {
                        echo "ℹ️ Không có thay đổi. Bỏ qua test & build."
                        SERVICES_CHANGED = ""
                        return
                    }

                    // Chuyển danh sách file thay đổi thành mảng
                    def changedFiles = changes.split("\n")

                    // Chuẩn hóa lại các đường dẫn của file để phù hợp với tên các dịch vụ
                    def normalizedChanges = changedFiles.collect { file ->
                        file.replaceFirst("^.*?/spring-petclinic-microservices/", "")
                    }

                    echo "✅ Các file thay đổi sau khi chuẩn hóa: ${normalizedChanges.join(', ')}"

                    // Danh sách các dịch vụ có sẵn
                    def services = [
                        "spring-petclinic-admin-server",
                        "spring-petclinic-api-gateway",
                        "spring-petclinic-config-server",
                        "spring-petclinic-customers-service",
                        "spring-petclinic-discovery-server",
                        "spring-petclinic-genai-service",
                        "spring-petclinic-vets-service",
                        "spring-petclinic-visits-service",
                    ]

                    // Lọc ra các dịch vụ có thay đổi
                    def changedServices = services.findAll { service ->
                        normalizedChanges.any { file ->
                            file.startsWith("${service}/") || file.contains("${service}/")
                        }
                    }

                    echo "📢 Các dịch vụ thay đổi: ${changedServices.join(', ')}"

                    // Nếu không có dịch vụ nào thay đổi, bỏ qua bước build
                    if (changedServices.isEmpty()) {
                        echo "ℹ️ Không có dịch vụ thay đổi. Bỏ qua test & build."
                        SERVICES_CHANGED = ""
                        return
                    }

                    // Lưu thông tin về các dịch vụ thay đổi vào properties của pipeline
                    properties([
                        parameters([
                            string(name: 'SERVICES_CHANGED', defaultValue: changedServices.join(','), description: 'Các dịch vụ thay đổi trong build này')
                        ])
                    ])

                    SERVICES_CHANGED = changedServices.join(',')
                    echo "🚀 Dịch vụ thay đổi (Toàn bộ môi trường): ${SERVICES_CHANGED}"
                }
            }
        }
        
        stage('Build (Maven)') {
            when {
                expression { SERVICES_CHANGED?.trim() != "" }
            }
            steps {
                script {
                    def servicesList = SERVICES_CHANGED.tokenize(',')

                    if (servicesList.isEmpty()) {
                        echo "ℹ️ Không có dịch vụ thay đổi. Bỏ qua build."
                        return
                    }

                    // Build từng dịch vụ thay đổi bằng Maven
                    for (service in servicesList) {
                        echo "🏗️ Đang build ${service}..."
                        dir(service) {
                            sh '../mvnw package -DskipTests -T 1C'
                        }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            when {
                expression { SERVICES_CHANGED?.trim() != "" }
            }
            steps {
                script {
                    def servicesList = SERVICES_CHANGED.tokenize(',')

                    if (servicesList.isEmpty()) {
                        error("❌ Không có dịch vụ thay đổi. Kiểm tra lại bước 'Detect Changes'.")
                    }

                    // Danh sách cổng cho từng dịch vụ
                    def servicePorts = [
                        "spring-petclinic-admin-server": 9090,
                        "spring-petclinic-api-gateway": 8080,
                        "spring-petclinic-config-server": 8888,
                        "spring-petclinic-customers-service": 8081,
                        "spring-petclinic-discovery-server": 8761,
                        "spring-petclinic-genai-service": 8084,
                        "spring-petpetclinic-vets-service": 8083,
                        "spring-petclinic-visits-service": 8082
                    ]

                    // Đăng nhập Docker Hub một lần trước khi bắt đầu build
                    withCredentials([usernamePassword(
                        credentialsId: 'docker_hub_PAT',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        echo "🔐 Đăng nhập vào Docker Hub..."
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    }

                    // Build và push Docker image cho từng dịch vụ thay đổi
                    for (service in servicesList) {
                        echo "🐳 Đang build & push Docker image cho ${service}..."

                        // Lấy tên dịch vụ ngắn
                        def shortServiceName = service.replaceFirst("spring-petclinic-", "")
                        
                        // Lấy cổng của dịch vụ
                        def servicePort = servicePorts.get(service, 8080)
                        
                        // Lấy commit hash
                        def commitHash = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        def imageTag = "hzeroxium/${service}:${commitHash}"

                        sh """
                        docker build \\
                            --build-arg SERVICE_NAME=${shortServiceName} \\
                            --build-arg EXPOSED_PORT=${servicePort} \\
                            -f Dockerfile \\
                            -t ${imageTag} \\
                            -t hzeroxium/${service}:latest \\
                            .
                        docker push ${imageTag}
                        docker push hzeroxium/${service}:latest
                        docker rmi ${imageTag} || true
                        docker rmi hzeroxium/${service}:latest || true
                        """
                    }
                }
            }
        }

        // stage('Update GitOps Repository') {
        //     when {
        //         expression { SERVICES_CHANGED?.trim() != "" }
        //     }
        //     steps {
        //         script {
        //             def servicesList = SERVICES_CHANGED.tokenize(',')
        //             def commitHash = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

        //             sh "rm -rf spring-petclinic-microservices-config || true"
                    
        //             withCredentials([usernamePassword(
        //                 credentialsId: 'github-credentials', 
        //                 usernameVariable: 'GIT_USERNAME', 
        //                 passwordVariable: 'GIT_PASSWORD'
        //             )]) {
        //                 sh """
        //                 git clone https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com/HZeroxium/spring-petclinic-microservices-config.git
        //                 """
                        
        //                 dir('spring-petclinic-microservices-config') {
        //                     for (service in servicesList) {
        //                         def shortServiceName = service.replaceFirst("spring-petclinic-", "")
        //                         def valuesFile = "values/dev/values-${shortServiceName}.yaml"
                                
        //                         sh """
        //                         if [ -f "${valuesFile}" ]; then
        //                             echo "Cập nhật tag image trong ${valuesFile}"
        //                             sed -i 's/\\(tag:\\s*\\).*/\\1"'${commitHash}'"/' ${valuesFile}
        //                         else
        //                             echo "Cảnh báo: ${valuesFile} không tìm thấy"
        //                         fi
        //                         """
        //                     }
                            
        //                     sh """
        //                     git config user.email "jenkins@example.com"
        //                     git config user.name "Jenkins CI"
        //                     git status
                            
        //                     if ! git diff --quiet; then
        //                         git add .
        //                         git commit -m "Cập nhật tag image cho ${SERVICES_CHANGED} thành ${commitHash}"
        //                         git push
        //                         echo "✅ Đã cập nhật thành công repository GitOps"
        //                     else
        //                         echo "ℹ️ Không có thay đổi nào cần commit trong repository GitOps"
        //                     fi
        //                     """
        //                 }
        //             }
                    
        //             sh "rm -rf spring-petclinic-microservices-config || true"
        //         }
        //     }
        // }
    }

    post {
        failure {
            script {
                echo "❌ Pipeline CI/CD thất bại!"
            }
        }
        success {
            script {
                echo "✅ Pipeline CI/CD thành công!"
            }
        }
        always {
            echo "✅ Pipeline hoàn thành cho các dịch vụ: ${SERVICES_CHANGED}"
        }
    }
}
