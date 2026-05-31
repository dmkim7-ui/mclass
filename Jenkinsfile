pipeline {
    agent any // 어떤 에이전트 (실행서버)에서든 실행 가능

    tools {
        maven 'maven 3.9.12' // Jenkins에 등록된 Maven 3.9.12을 사용
    }

    stages {
        stage('Git Checkout') {
            steps { // steps : stage 안에 실행할 실제 코드
                // Jenkins에 연결된 Git 저장소에서 최신 코드 체크 아웃}
                checkout scm
            }
        }
    }
}