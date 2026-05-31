pipeline {
    agent any // 어떤 에이전트 (실행서버)에서든 실행 가능

    tools {
        maven 'maven 3.9.12' // Jenkins에 등록된 Maven 3.9.12을 사용
    }

    environment {
        DOCKER_IMAGE = "damo-app"
        CONTAINER_NAME = "springboot-container"
        JAR_FILE_NAME = "app.jar"
        PORT = "8081"
        REMOTE_USER = "ec2-user"
        REMOTE_HOST = "ec2-3-36-123-189.ap-northeast-2.compute.amazonaws.com"
        REMOTE_DIR = "/home/ec2-user/deploy"

        SSH_CREDENTIALS_ID = "279b6f82-58f7-41ae-bb10-7d1b8c729b06"
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Prepare Jar') {
            steps {
                sh 'cp target/demo-0.0.1-SNAPSHOT.jar ${JAR_FILE_NAME}'
            }
        }

        stage('Copy to Remote Server') {
            steps {
                // 파일 전송 후 세션 종료 버그가 빌드를 멈추지 않도록 방어막 배치
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_DIR}"
                        scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${JAR_FILE_NAME} Dockerfile ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/
                        """
                    }
                }
            }
        }

        stage('Remote Docker Build * Deploy') {
            steps {
                // 도커 빌드 후 세션 종료 버그가 빌드를 멈추지 않도록 방어막 배치
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} "
                            cd ${REMOTE_DIR} && \
                            docker rm -f ${CONTAINER_NAME} 2>/dev/null || true && \
                            docker build -t ${DOCKER_IMAGE} . && \
                            docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} ${DOCKER_IMAGE}
                        "
                        """
                    }
                }
            }
        }     

    }
}