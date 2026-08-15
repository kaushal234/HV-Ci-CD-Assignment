def FAILED_STAGE = ''

pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '465708537536.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO = 'hv-ci-cd-assignment'
        IMAGE_TAG = "${GIT_COMMIT.take(7)}"
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
        APP_EC2_HOST = '54.84.160.10'
        APP_EC2_USER = 'ec2-user'
        CONTAINER_NAME = 'student-app'
        NOTIFY_EMAIL = 'kaushal.hirani.sde@gmail.com'
    }

    stages {
        stage('Checkout') {
            steps {
                script { FAILED_STAGE = 'Checkout' }
                checkout scm
            }
        }

        stage('Install') {
            steps {
                script { FAILED_STAGE = 'Install' }
                sh 'python3 -m pip install --user -r requirements.txt'
            }
        }

        stage('Test') {
            steps {
                script { FAILED_STAGE = 'Test' }
                sh '''
                    docker run -d --name test-mongo -p 27017:27017 mongo:latest
                    for i in $(seq 1 30); do
                        if docker exec test-mongo mongosh --eval "db.runCommand({ping:1})" >/dev/null 2>&1; then
                            break
                        fi
                        sleep 2
                    done
                    MONGO_URI="mongodb://localhost:27017/test_student_db" SECRET_KEY="test" python3 -m pytest -v
                '''
            }
            post {
                always {
                    sh 'docker rm -f test-mongo || true'
                }
            }
        }

        stage('Build') {
            steps {
                script { FAILED_STAGE = 'Build' }
                sh 'docker build -t ${IMAGE_URI} .'
            }
        }

        stage('Push to ECR') {
            steps {
                script { FAILED_STAGE = 'Push to ECR' }
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${IMAGE_URI}
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script { FAILED_STAGE = 'Deploy to EC2' }
                withCredentials([
                    string(credentialsId: 'MONGO_URI', variable: 'MONGO_URI'),
                    string(credentialsId: 'SECRET_KEY', variable: 'SECRET_KEY')
                ]) {
                    sshagent(['app-ec2-ssh-key']) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} "
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY} &&
                                docker pull ${IMAGE_URI} &&
                                docker stop ${CONTAINER_NAME} || true &&
                                docker rm ${CONTAINER_NAME} || true &&
                                docker run -d --restart unless-stopped --name ${CONTAINER_NAME} -p 5000:5000 -e MONGO_URI='${MONGO_URI}' -e SECRET_KEY='${SECRET_KEY}' ${IMAGE_URI}
                            "
                        '''
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                script { FAILED_STAGE = 'Verify' }
                sshagent(['app-ec2-ssh-key']) {
                    sh '''
                        sleep 5
                        ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} "
                            curl -f http://localhost:5000/health
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "[SUCCESS] ${JOB_NAME} #${BUILD_NUMBER} deployed",
                body: """The CI/CD pipeline succeeded.

Commit SHA: ${GIT_COMMIT}
Branch: ${GIT_BRANCH}
Image tag: ${IMAGE_TAG}
Image: ${IMAGE_URI}
Deployed to EC2: ${APP_EC2_HOST}

Pipeline run: ${BUILD_URL}
"""
            )
        }
        failure {
            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "[FAILURE] ${JOB_NAME} #${BUILD_NUMBER} failed at ${FAILED_STAGE}",
                body: """The CI/CD pipeline failed.

Failed stage: ${FAILED_STAGE}
Commit SHA: ${GIT_COMMIT}
Branch: ${GIT_BRANCH}

Logs: ${BUILD_URL}console
"""
            )
        }
    }
}