pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'complaintbox'
        ECR_REGISTRY = '104360117424.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_REPO = 'https://github.com/shaunjohn-04/ComplaintBox-DevSecOps.git'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('Bandit - SAST') {
            steps {
                sh '''
                    mkdir -p reports
                    docker run --rm \
                    -v "$WORKSPACE:/src" \
                    -w /src python:3.11-slim \
                    sh -c "pip install --quiet bandit &&
                    bandit -r complaint_box -f json \
                    -o reports/bandit-report.json || true"
                '''
            }
        }

        stage('Dependency Scan - pip-audit') {
            steps {
                sh '''
                    docker run --rm \
                    -v "$WORKSPACE:/src" \
                    -w /src python:3.11-slim \
                    sh -c "apt-get update -qq &&
                    apt-get install -y -qq gcc pkg-config default-libmysqlclient-dev &&
                    pip install --quiet pip-audit &&
                    pip-audit -r requirements.txt \
                    -f json -o reports/pip-audit-report.json || true"
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    docker run --rm \
                    --network host \
                    -v "$WORKSPACE/reports:/zap/wrk/:rw" \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                    -t http://127.0.0.1:8000 \
                    -m 1 -T 120 \
                    -r zap-report.html || true
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t complaintbox:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push to AWS ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',
                           variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key',
                           variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION=$AWS_REGION

                        aws ecr get-login-password \
                        --region $AWS_REGION | \
                        docker login \
                        --username AWS \
                        --password-stdin $ECR_REGISTRY

                        docker tag complaintbox:${BUILD_NUMBER} \
                        $ECR_REGISTRY/$ECR_REPO:${BUILD_NUMBER}

                        docker push \
                        $ECR_REGISTRY/$ECR_REPO:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                sh '''
                    sed -i "s|image: .*|image: ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}|" \
                    k8s/deployment.yaml

                    echo "Updated Kubernetes image:"
                    grep "image:" k8s/deployment.yaml
                '''
            }
        }

        stage('Push Manifest to GitHub') {
            steps {
                withCredentials([
                    string(credentialsId: 'github-token',
                           variable: 'GITHUB_TOKEN')
                ]) {
                    sh '''
                        set +x

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add k8s/deployment.yaml

                        git commit -m "Update ComplaintBox image to build ${BUILD_NUMBER}" \
                        || echo "No changes to commit"

                        AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w 0)

                        git -c http.extraheader="AUTHORIZATION: basic $AUTH" \
                        push origin HEAD:main

                        unset AUTH GITHUB_TOKEN
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**',
                             allowEmptyArchive: true
        }

        success {
            echo 'ComplaintBox CI/CD Pipeline SUCCESS!'
            echo 'GitHub manifest updated. Argo CD will deploy the new image.'
        }

        failure {
            echo 'ComplaintBox Pipeline FAILED. Check console output.'
        }
    }
}
