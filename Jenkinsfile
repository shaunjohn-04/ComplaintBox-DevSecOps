pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'complaintbox'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/shaunjohn-04/ComplaintBox-DevSecOps.git'
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
                    bandit -r complaint_box -f json -o reports/bandit-report.json || true"
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
                    pip-audit -r requirements.txt -f json -o reports/pip-audit-report.json || true"
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

                    if [ -f reports/zap-report.html ]; then
                        echo "ZAP report generated"
                    fi
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t complaintbox:${BUILD_NUMBER} .
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

                        ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account --output text)

                        aws ecr describe-repositories \
                            --repository-names $ECR_REPO \
                            >/dev/null 2>&1 || \
                        aws ecr create-repository \
                            --repository-name $ECR_REPO

                        aws ecr get-login-password \
                            --region $AWS_REGION | \
                        docker login \
                            --username AWS \
                            --password-stdin \
                            $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                        docker tag complaintbox:${BUILD_NUMBER} \
                            $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:${BUILD_NUMBER}

                        docker push \
                            $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:${BUILD_NUMBER}
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
            echo 'ComplaintBox DevSecOps Pipeline SUCCESS!'
        }

        failure {
            echo 'ComplaintBox Pipeline FAILED. Check console output.'
        }
    }
}
