pipeline {
    agent any
    environment {
        DOCKER_REPO = 'lerisngwa/landmark-web-app'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'landmark-eks-cluster'
    }
    options {
        skipDefaultCheckout(false)
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install & Test') {
            agent { docker { image 'node:18-alpine'; args '-v /var/lib/jenkins/.npm:/.npm' } }
            steps {
                sh 'rm -rf node_modules server/node_modules'
                sh 'npm install --cache /.npm'
                sh 'npm test -- --watchAll=false'
                sh 'cd server && npm install --cache /.npm && npm test'
            }
        }
        stage('Build Frontend') {
            agent { docker { image 'node:18-alpine'; args '-v /var/lib/jenkins/.npm:/.npm' } }
            steps {
                sh 'npm run build --cache /.npm'
            }
        }
        stage('Generate Image Tag') {
            steps {
                script {
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                }
            }
        }
        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'staging'
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                sh """
                    echo \$DOCKERHUB_PASSWORD | docker login -u \$DOCKERHUB_USERNAME --password-stdin
                    docker build -t ${DOCKER_REPO}:${IMAGE_TAG} .
                    docker push ${DOCKER_REPO}:${IMAGE_TAG}
                """
            }
        }
        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                    sed -i 's/name: landmark/name: develop/g' k8s/namespace.yml
                    kubectl apply -f k8s/namespace.yml --validate=false
                    sed -i 's/namespace: landmark/namespace: develop/g' k8s/*.yml
                    sed -i "s|image: landmark-technologies:latest.*|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                    kubectl apply -f k8s/ --validate=false
                """
            }
        }
        stage('Deploy to Staging') {
            when { branch 'staging' }
            steps {
                sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                    sed -i 's/name: landmark/name: staging/g' k8s/namespace.yml
                    kubectl apply -f k8s/namespace.yml --validate=false
                    sed -i 's/namespace: landmark/namespace: staging/g' k8s/*.yml
                    sed -i "s|image: landmark-technologies:latest.*|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                    kubectl apply -f k8s/ --validate=false
                """
            }
        }
        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                    sed -i 's/name: landmark/name: production/g' k8s/namespace.yml
                    kubectl apply -f k8s/namespace.yml --validate=false
                    sed -i 's/namespace: landmark/namespace: production/g' k8s/*.yml
                    sed -i "s|image: landmark-technologies:latest.*|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                    kubectl apply -f k8s/ --validate=false
                """
            }
        }
    }
    post {
        failure { echo 'Pipeline failed!' }
        success { echo 'Pipeline succeeded!' }
    }
}
