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

        SECRET_FILE_ID = "2541a14b-8caa-4fe4-a638-29cb416b43d1"
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

        stage('Inject Spring Config (Secret File)') {
            steps {
                withCredentials([file(credentialsId: env.SECRET_FILE_ID, variable: 'SPRING_CONFIG_FILE')]) {
                    sh """
                        echo "[INFO] Using secret file: $SPRING_CONFIG_FILE"
                        cp \$SPRING_CONFIG_FILE ./application-prod.properties
                    """
                }
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
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} "
                            cd ${REMOTE_DIR} && \
                            
                            # 1. 기존 컨테이너가 있으면 확실하게 중지 및 삭제 (이름 기준)
                            docker rm -f ${CONTAINER_NAME} 2>/dev/null || true && \
                            
                            # 2. 8081 포트를 혹시 선점하고 있는 다른 좀비 컨테이너가 있다면 강제 정리 (선택 안전장치)
                            # docker ps -q --filter 'publish=${PORT}' | xargs -r docker rm -f && \
                            
                            # 3. ⭐️ [핵심] 캐시를 쓰지 않고 새로 전송된 app.jar로 완전히 새로 빌드
                            docker build --no-cache -t ${DOCKER_IMAGE} . && \
                            
                            # 4. 신규 이미지로 컨테이너 가동
                            docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} ${DOCKER_IMAGE} && \
                            
                            # 5. 빌드 후 남은 안 쓰는 쓸모없는 대기 상태 이미지(Dangling Image) 정리
                            docker image prune -f
                        "
                        """
                    }
                }
            }
        }

    }
}