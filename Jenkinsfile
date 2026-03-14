pipeline {
    agent any
    
    stages {
        // --- STAGE 1: QUÉT BẢO MẬT TỔNG THỂ ---
        stage('Global Security Scan') {
            steps {
                script {
                    echo '=== 1.1 Quét lộ mật khẩu (Gitleaks) ==='
                    docker.image('zricethezav/gitleaks:latest').inside('--entrypoint=""') {
                        sh 'gitleaks detect --source="." --no-git --verbose || true'
                    }
                }
            }
        }

        // --- STAGE 2: CHUẨN BỊ POM GỐC ---
        stage('Prepare Root & Commons') {
            steps {
                echo '=== Cài đặt cấu hình gốc và thư viện dùng chung ==='
                script {
                    // FIX: Đổi từ thư mục vật lý sang Named Volume 'yas-m2-cache' để tránh lỗi quyền ghi
                    docker.image('maven:3.9.6-eclipse-temurin-21').inside("-v yas-m2-cache:/m2repo") {
                        // FIX: Dẫn repo vào đúng volume /m2repo
                        sh 'mvn install -N -Drevision=1.0-SNAPSHOT -Dmaven.repo.local=/m2repo'
                        sh 'mvn clean install -DskipTests -Drevision=1.0-SNAPSHOT -pl common-library -am -Dmaven.repo.local=/m2repo'
                    }
                }
            }
        }

        // --- STAGE 3: QUY TRÌNH CI TUẦN TỰ ---
        stage('Business Services CI') {
            stages {
                stage('Service: Customer') { when { changeset "customer/**" }; steps { runServiceCI('customer') } }
                stage('Service: Product') { when { changeset "product/**" }; steps { runServiceCI('product') } }
                stage('Service: Cart') { when { changeset "cart/**" }; steps { runServiceCI('cart') } }
                stage('Service: Order') { when { changeset "order/**" }; steps { runServiceCI('order') } }
                stage('Service: Media') { when { changeset "media/**" }; steps { runServiceCI('media') } }
                stage('Service: Rating') { when { changeset "rating/**" }; steps { runServiceCI('rating') } }
                stage('Service: Location') { when { changeset "location/**" }; steps { runServiceCI('location') } }
                stage('Service: Inventory') { when { changeset "inventory/**" }; steps { runServiceCI('inventory') } }
                stage('Service: Tax') { when { changeset "tax/**" }; steps { runServiceCI('tax') } }
                stage('Service: Search') { when { changeset "search/**" }; steps { runServiceCI('search') } }
                stage('Service: Payment') { when { changeset "payment/**" }; steps { runServiceCI('payment') } }
                stage('Service: Promotion') { when { changeset "promotion/**" }; steps { runServiceCI('promotion') } }
                stage('Service: Backoffice-BFF') { when { changeset "backoffice-bff/**" }; steps { runServiceCI('backoffice-bff') } }
                stage('Service: Storefront-BFF') { when { changeset "storefront-bff/**" }; steps { runServiceCI('storefront-bff') } }
                stage('Service: Sampledata') { when { changeset "sampledata/**" }; steps { runServiceCI('sampledata') } }
                stage('Service: payment-paypal') { when { changeset "payment-paypal/**" }; steps { runServiceCI('payment-paypal') } }
            }
        }
    }

    post {
        always {
            script {
                echo "=== BẮT ĐẦU DỌN DẸP TÀI NGUYÊN (RESET) ==="
                sh 'docker ps -aq --filter label=org.testcontainers=true | xargs -r docker rm -f || true'
                sh 'docker network prune -f || true'
            }
        }
    }
}

// --- HÀM HỖ TRỢ XỬ LÝ TỪNG SERVICE ---
def runServiceCI(String serviceName) {
    script {
        // FIX: Phải dùng chung Named Volume 'yas-m2-cache' như ở Stage 2
        docker.image('maven:3.9.6-eclipse-temurin-21').inside("-v yas-m2-cache:/m2repo") {
            echo "=== Phase: Unit Test & Sonar Scan cho ${serviceName} ==="
            
            withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                // FIX: Trỏ repo vào /m2repo
                sh """mvn install sonar:sonar \
                -Drevision=1.0-SNAPSHOT -pl ${serviceName} -am \
                -DskipITs=true \
                -Dmaven.repo.local=/m2repo \
                -Dsonar.token=\$SONAR_TOKEN \
                -Dsonar.organization=longlee0 \
                -Dsonar.projectKey=LongLee0_yas_Project1_Devops"""
            }
            
            echo "=== Phase: Kiểm tra độ phủ Test > 70% (Yêu cầu 7b) ==="
            jacoco(
                execPattern: "${serviceName}/target/*.exec",
                classPattern: "${serviceName}/target/classes",
                sourcePattern: "${serviceName}/src/main/java",
                inclusionPattern: "**/*.class",
                minimumInstructionCoverage: '70',
                maximumInstructionCoverage: '70',
                buildOverBuild: false,
                changeBuildStatus: true,
                skipCopyOfSrcFiles: true 
            )
        }

        echo "=== Phase: Build Docker Image cho ${serviceName} ==="
        dir(serviceName) {
            sh "rm -f target/*-tests.jar target/*.jar.original || true"
            sh "docker build -t yas-${serviceName}:${BUILD_ID} ."
        }
    }
    publishTestResults(serviceName)
}

def publishTestResults(String serviceName) {
    dir(serviceName) {
        junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
    }
}