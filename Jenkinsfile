pipeline {
    agent any

    environment {
        SCANNER_HOME      = tool 'SonarScanner'

        MASTER_IP         = '52.90.99.154'
        NEXUS_URL         = "52.90.99.154:8082"
        NEXUS_REPO        = 'docker-hosted'

        IMAGE_NAME        = 'myapp'
        IMAGE_TAG         = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'}"
        DOCKER_IMAGE      = "${NEXUS_URL}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
        DOCKER_IMAGE_LATEST = "${NEXUS_URL}/${NEXUS_REPO}/${IMAGE_NAME}:latest"

        SONAR_PROJECT_KEY = 'my-app'
        SONAR_HOST_URL    = "http://52.90.99.154:9000"

        GIT_REPO_URL      = 'https://github.com/omkar0046/CI-CD-Automation.git'
        GIT_BRANCH        = 'main'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'qa', 'prod'], description: 'Select deployment environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test execution')
        booleanParam(name: 'SKIP_SONAR', defaultValue: false, description: 'Skip SonarQube analysis')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: 'git-credentials',
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                    chmod +x mvnw 2>/dev/null || true

                    if [ -f "mvnw" ]; then
                        ./mvnw clean package -DskipTests=${SKIP_TESTS}
                    elif [ -f "pom.xml" ]; then
                        mvn clean package -DskipTests=${SKIP_TESTS}
                    elif [ -f "build.gradle" ]; then
                        chmod +x gradlew
                        ./gradlew clean build -x test
                    elif [ -f "package.json" ]; then
                        npm ci
                        npm run build
                    else
                        echo "No build file found, skipping build step"
                    fi
                '''
            }
        }

        stage('Unit Tests') {
            when { expression { !params.SKIP_TESTS } }
            steps {
                sh '''
                    if [ -f "pom.xml" ]; then
                        mvn test
                    elif [ -f "package.json" ]; then
                        npm test || true
                    fi
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { !params.SKIP_SONAR } }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName=${IMAGE_NAME} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=${SONAR_TOKEN} \
                          -Dsonar.sourceEncoding=UTF-8
                    """
                }
            }
        }

        stage('Quality Gate') {
            when { expression { !params.SKIP_SONAR } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        --output trivy-fs-report.txt \
                        .
                '''
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} -t ${DOCKER_IMAGE_LATEST} .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --format table \
                        --output trivy-image-report.txt \
                        ${DOCKER_IMAGE}
                """
                archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
            }
        }

        stage('Docker Login & Push to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo "${NEXUS_PASS}" | docker login http://${NEXUS_URL} -u "${NEXUS_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker push ${DOCKER_IMAGE_LATEST}
                        docker logout http://${NEXUS_URL}
                    """
                }
            }
        }

        stage('Create Namespace') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        kubectl --kubeconfig=${KUBECONFIG_FILE} create namespace ${params.DEPLOY_ENV} --dry-run=client -o yaml | \
                        kubectl --kubeconfig=${KUBECONFIG_FILE} apply -f -
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        sed -i "s|DOCKER_IMAGE_PLACEHOLDER|${DOCKER_IMAGE}|g" k8s/deployment.yaml

                        kubectl --kubeconfig=${KUBECONFIG_FILE} apply -f k8s/deployment.yaml -n ${params.DEPLOY_ENV}

                        kubectl --kubeconfig=${KUBECONFIG_FILE} rollout status deployment/${IMAGE_NAME} -n ${params.DEPLOY_ENV} --timeout=180s
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        kubectl --kubeconfig=${KUBECONFIG_FILE} get pods -n ${params.DEPLOY_ENV} -o wide
                        kubectl --kubeconfig=${KUBECONFIG_FILE} get svc -n ${params.DEPLOY_ENV}
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${DOCKER_IMAGE} 2>/dev/null || true
                docker rmi ${DOCKER_IMAGE_LATEST} 2>/dev/null || true
                docker image prune -f 2>/dev/null || true
            """
            cleanWs()
        }

        success {
            echo "✅ Pipeline SUCCESS - Deployed to ${params.DEPLOY_ENV}"
        }

        failure {
            echo "❌ Pipeline FAILED - Check logs"
        }
    }
}
