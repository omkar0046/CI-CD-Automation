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
        booleanParam(name: 'SKIP_TESTS', defaultValue: true, description: 'Skip test execution')
        booleanParam(name: 'SKIP_SONAR', defaultValue: false, description: 'Skip SonarQube analysis')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Cleanup Workspace') {
            steps { cleanWs() }
        }

        stage('Git Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: 'git-credentials',
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('SonarQube Analysis (Fast Mode)') {
            when { expression { !params.SKIP_SONAR } }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=my-app \
                          -Dsonar.projectName=myapp \
                          -Dsonar.sources=k8s \
                          -Dsonar.inclusions=**/Dockerfile,**/Jenkinsfile,**/*.yaml \
                          -Dsonar.exclusions=**/node_modules/**,**/target/**,**/.git/** \
                          -Dsonar.host.url=http://52.90.99.154:9000 \
                          -Dsonar.token=$SONAR_TOKEN \
                          -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} -t ${DOCKER_IMAGE_LATEST} .
                """
            }
        }

        stage('Docker Push to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        echo "$NEXUS_PASS" | docker login 52.90.99.154:8082 -u "$NEXUS_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker push ${DOCKER_IMAGE_LATEST}
                        docker logout 52.90.99.154:8082
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        kubectl --kubeconfig=$KUBECONFIG_FILE create namespace ${DEPLOY_ENV} --dry-run=client -o yaml | kubectl --kubeconfig=$KUBECONFIG_FILE apply -f -

                        sed -i "s|DOCKER_IMAGE_PLACEHOLDER|${DOCKER_IMAGE}|g" k8s/deployment.yaml

                        kubectl --kubeconfig=$KUBECONFIG_FILE apply -f k8s/deployment.yaml -n ${DEPLOY_ENV}

                        kubectl --kubeconfig=$KUBECONFIG_FILE rollout status deployment/myapp -n ${DEPLOY_ENV} --timeout=120s
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rmi ${DOCKER_IMAGE} 2>/dev/null || true
                docker rmi ${DOCKER_IMAGE_LATEST} 2>/dev/null || true
                docker image prune -f 2>/dev/null || true
            '''
            cleanWs()
        }

        success {
            echo "✅ FAST PIPELINE SUCCESS - Deployed to ${params.DEPLOY_ENV}"
        }

        failure {
            echo "❌ Pipeline FAILED"
        }
    }
}
