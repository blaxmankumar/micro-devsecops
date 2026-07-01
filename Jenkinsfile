pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        REPO_URL = 'https://github.com/blaxmankumar/micro-devsecops.git'

        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_PROJECT_KEY = 'micro-devsecops'

        NEXUS_URL = 'http://localhost:8081'
        NEXUS_REPO = 'devscops'

        DOCKERHUB_NAMESPACE = 'battulalaxmankumar04'
        IMAGE_TAG = "${BUILD_NUMBER}"

        RUN_PIPELINE = 'true'
    }

    stages {

        stage('Clone Code') {
            steps {
                script {
                    if (env.BRANCH_NAME) {
                        checkout scm
                    } else {
                        git branch: 'main', url: "${REPO_URL}"
                    }
                }
            }
        }

        stage('Check Changed Files') {
            steps {
                script {
                    def changedFiles = sh(
                        script: '''
                        git diff --name-only HEAD~1 HEAD 2>/dev/null || git show --name-only --pretty="" HEAD
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Changed files:"
                    echo changedFiles

                    def importantChanges = changedFiles
                        .split('\\n')
                        .findAll { file ->
                            file &&
                            !file.startsWith('k8s/') &&
                            !file.startsWith('corrected-ecommerce-helm-charts/')
                        }

                    if (importantChanges.isEmpty()) {
                        echo "Only k8s/ or Helm manifest files changed."
                        echo "Skipping CI pipeline to avoid loop."
                        env.RUN_PIPELINE = 'false'
                        currentBuild.result = 'NOT_BUILT'
                    } else {
                        echo "Application/source code changes found."
                        echo "Running full DevSecOps pipeline."
                        env.RUN_PIPELINE = 'true'
                    }
                }
            }
        }

        stage('Start SonarQube and Nexus') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                sh '''
                set +e

                if [ -d "sonar-nexus" ]; then
                    cd sonar-nexus

                    echo "Checking SonarQube and Nexus containers..."

                    RUNNING_SERVICES=$(docker compose ps --services --filter "status=running" || true)

                    echo "Running services:"
                    echo "$RUNNING_SERVICES"

                    SONAR_RUNNING=$(echo "$RUNNING_SERVICES" | grep -w "sonarqube" || true)
                    NEXUS_RUNNING=$(echo "$RUNNING_SERVICES" | grep -w "nexus" || true)

                    if [ -n "$SONAR_RUNNING" ] && [ -n "$NEXUS_RUNNING" ]; then
                        echo "SonarQube and Nexus are already running."
                    else
                        echo "Starting SonarQube and Nexus..."
                        docker compose up -d
                        echo "Waiting for SonarQube and Nexus..."
                        sleep 120
                    fi

                    cd ..
                else
                    echo "sonar-nexus folder not found. Skipping docker compose startup."
                fi

                echo "Running containers:"
                docker ps

                echo "Checking SonarQube status..."
                curl -f ${SONAR_HOST_URL}/api/system/status || true

                echo "Checking Nexus status..."
                curl -I ${NEXUS_URL} || true
                '''
            }
        }

        stage('SonarQube Scan') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                    set -e

                    mkdir -p reports/sonarqube .scannerwork .sonar/cache
                    chmod -R 777 reports .scannerwork .sonar || true

                    docker run --rm \
                      --network host \
                      -e SONAR_HOST_URL="${SONAR_HOST_URL}" \
                      -e SONAR_TOKEN="${SONAR_TOKEN}" \
                      -v "$PWD:/usr/src" \
                      -v "$PWD/.sonar/cache:/opt/sonar-scanner/.sonar/cache" \
                      sonarsource/sonar-scanner-cli:latest \
                      -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
                      -Dsonar.projectName="${SONAR_PROJECT_KEY}" \
                      -Dsonar.sources=. \
                      -Dsonar.host.url="${SONAR_HOST_URL}" \
                      -Dsonar.token="${SONAR_TOKEN}" \
                      -Dsonar.java.binaries=. \
                      -Dsonar.exclusions="**/node_modules/**,**/dist/**,**/build/**,**/target/**,**/.gradle/**,**/.git/**,**/reports/**"

                    cp -r .scannerwork reports/sonarqube/scannerwork || true
                    '''
                }
            }
        }

        stage('Run JUnit Tests') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                sh '''
                set +e

                mkdir -p reports/junit

                MAVEN_SERVICES="api-gateway user-service order-service inventory-service"
                GRADLE_SERVICES="product-service cart-service payment-service notification-service"

                for service in $MAVEN_SERVICES; do
                    if [ -d "$service" ]; then
                        echo "Running Maven tests for $service"
                        cd $service
                        mvn test
                        TEST_STATUS=$?
                        cd ..

                        mkdir -p reports/junit/$service
                        cp -r $service/target/surefire-reports/* reports/junit/$service/ 2>/dev/null || true

                        if [ $TEST_STATUS -ne 0 ]; then
                            echo "Tests failed for $service, but continuing so reports can be collected."
                        fi
                    else
                        echo "$service folder not found. Skipping."
                    fi
                done

                for service in $GRADLE_SERVICES; do
                    if [ -d "$service" ]; then
                        echo "Running Gradle tests for $service"
                        cd $service
                        chmod +x gradlew 2>/dev/null || true

                        if [ -f "./gradlew" ]; then
                            ./gradlew test
                        else
                            gradle test
                        fi

                        TEST_STATUS=$?
                        cd ..

                        mkdir -p reports/junit/$service
                        cp -r $service/build/test-results/test/* reports/junit/$service/ 2>/dev/null || true

                        if [ $TEST_STATUS -ne 0 ]; then
                            echo "Tests failed for $service, but continuing so reports can be collected."
                        fi
                    else
                        echo "$service folder not found. Skipping."
                    fi
                done

                tar -czf junit-reports.tar.gz reports/junit || true
                '''
            }

            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit/**/*.xml'
                    archiveArtifacts artifacts: 'reports/junit/**', allowEmptyArchive: true
                }
            }
        }

        stage('Build Application Artifacts') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                sh '''
                set +e

                MAVEN_SERVICES="api-gateway user-service order-service inventory-service"
                GRADLE_SERVICES="product-service cart-service payment-service notification-service"

                for service in $MAVEN_SERVICES; do
                    if [ -d "$service" ]; then
                        echo "Building Maven service: $service"
                        cd $service
                        mvn clean package -DskipTests
                        cd ..
                    fi
                done

                for service in $GRADLE_SERVICES; do
                    if [ -d "$service" ]; then
                        echo "Building Gradle service: $service"
                        cd $service
                        chmod +x gradlew 2>/dev/null || true

                        if [ -f "./gradlew" ]; then
                            ./gradlew clean build -x test
                        else
                            gradle clean build -x test
                        fi

                        cd ..
                    fi
                done
                '''
            }
        }

        stage('Upload JUnit Reports to Nexus') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                    if [ -f junit-reports.tar.gz ]; then
                        curl -u "$NEXUS_USER:$NEXUS_PASS" \
                          --upload-file junit-reports.tar.gz \
                          "${NEXUS_URL}/repository/${NEXUS_REPO}/junit/junit-reports-${BUILD_NUMBER}.tar.gz"
                    else
                        echo "junit-reports.tar.gz not found. Skipping upload."
                    fi
                    '''
                }
            }
        }

        stage('Upload Sonar Report to Nexus') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                    tar -czf sonarqube-report.tar.gz reports/sonarqube || true

                    if [ -f sonarqube-report.tar.gz ]; then
                        curl -u "$NEXUS_USER:$NEXUS_PASS" \
                          --upload-file sonarqube-report.tar.gz \
                          "${NEXUS_URL}/repository/${NEXUS_REPO}/sonarqube/sonarqube-report-${BUILD_NUMBER}.tar.gz"
                    else
                        echo "sonarqube-report.tar.gz not found. Skipping upload."
                    fi
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                sh '''
                set -e

                services="api-gateway user-service product-service cart-service order-service payment-service inventory-service notification-service frontend"

                for service in $services; do
                    if [ -d "$service" ]; then
                        echo "Building Docker image for $service"

                        docker build \
                          -t ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG} \
                          -t ${DOCKERHUB_NAMESPACE}/$service:latest \
                          ./$service
                    else
                        echo "$service folder not found. Skipping image build."
                    fi
                done
                '''
            }
        }

        stage('Trivy Scan Images') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                sh '''
                set +e

                mkdir -p reports/trivy

                services="api-gateway user-service product-service cart-service order-service payment-service inventory-service notification-service frontend"

                for service in $services; do
                    if docker image inspect ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG} >/dev/null 2>&1; then
                        echo "Scanning image: ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG}"

                        trivy image \
                          --format table \
                          --output reports/trivy/$service-trivy-report.txt \
                          ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG} || true
                    else
                        echo "Image not found for $service. Skipping Trivy scan."
                    fi
                done

                tar -czf trivy-reports.tar.gz reports/trivy || true
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy/**', allowEmptyArchive: true
                }
            }
        }

        stage('Upload Trivy Reports to Nexus') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                    if [ -f trivy-reports.tar.gz ]; then
                        curl -u "$NEXUS_USER:$NEXUS_PASS" \
                          --upload-file trivy-reports.tar.gz \
                          "${NEXUS_URL}/repository/${NEXUS_REPO}/trivy/trivy-reports-${BUILD_NUMBER}.tar.gz"
                    else
                        echo "trivy-reports.tar.gz not found. Skipping upload."
                    fi
                    '''
                }
            }
        }

        stage('Push Images to Docker Hub') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    set -e

                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    services="api-gateway user-service product-service cart-service order-service payment-service inventory-service notification-service frontend"

                    for service in $services; do
                        if docker image inspect ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG} >/dev/null 2>&1; then
                            echo "Pushing ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG}"
                            docker push ${DOCKERHUB_NAMESPACE}/$service:${IMAGE_TAG}

                            echo "Pushing ${DOCKERHUB_NAMESPACE}/$service:latest"
                            docker push ${DOCKERHUB_NAMESPACE}/$service:latest
                        else
                            echo "Image not found for $service. Skipping push."
                        fi
                    done
                    '''
                }
            }
        }

        stage('Update K8s Image Tags in GitHub') {
            when {
                expression { env.RUN_PIPELINE == 'true' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                    set -e

                    TARGET_BRANCH="${BRANCH_NAME:-main}"

                    if [ -n "$CHANGE_BRANCH" ]; then
                        TARGET_BRANCH="$CHANGE_BRANCH"
                    fi

                    echo "Updating Kubernetes image tags to: ${IMAGE_TAG}"
                    echo "Target branch: ${TARGET_BRANCH}"

                    if [ ! -d "k8s" ]; then
                        echo "k8s folder not found. Skipping Kubernetes manifest update."
                        exit 0
                    fi

                    services="api-gateway user-service product-service cart-service order-service payment-service inventory-service notification-service frontend"

                    for service in $services; do
                        echo "Updating image for $service"

                        find k8s -type f \\( -name "*.yaml" -o -name "*.yml" \\) \
                          -exec sed -i -E "s#image:[[:space:]]*([A-Za-z0-9._-]+/)?${service}:[A-Za-z0-9._-]+#image: ${DOCKERHUB_NAMESPACE}/${service}:${IMAGE_TAG}#g" {} \\;

                        find k8s -type f \\( -name "*.yaml" -o -name "*.yml" \\) \
                          -exec sed -i -E "s#image:[[:space:]]*${DOCKERHUB_NAMESPACE}/${service}:[A-Za-z0-9._-]+#image: ${DOCKERHUB_NAMESPACE}/${service}:${IMAGE_TAG}#g" {} \\;
                    done

                    echo "Git status after image tag update:"
                    git status --short k8s

                    if git diff --quiet -- k8s; then
                        echo "No Kubernetes manifest image tag changes found. Nothing to commit."
                    else
                        git config user.email "jenkins@example.com"
                        git config user.name "jenkins-ci"

                        git add k8s
                        git commit -m "Update Kubernetes image tags to build ${IMAGE_TAG} [skip ci]"

                        git push "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/blaxmankumar/micro-devsecops.git" HEAD:${TARGET_BRANCH}

                        echo "Kubernetes image tags updated and pushed successfully."
                    fi
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }

        success {
            echo "DevSecOps pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Check the failed stage logs."
        }
    }
}
