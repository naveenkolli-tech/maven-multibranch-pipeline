pipeline {
    agent { label 'maven-agent' }

    environment {
        GITHUB_CREDENTIALS = 'YOUR_GITHUB_CREDENTIAL_ID'
        SSH_CREDENTIALS = 'YOUR_SSH_CREDENTIAL_ID'
        APP_SERVER = "ec2-user@APP_SERVER_IP"
        QA_SERVER = "ec2-user@QA_SERVER_IP"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                mvn org.owasp:dependency-check-maven:check \
                   -Dformat=HTML
                '''
            }
            post {
                failure {
                    error "Security scan failed — check vulnerabilities!"
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage('Deploy to QA') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh """
                    scp target/*.jar ${QA_SERVER}:/home/ec2-user/app.jar
                    ssh ${QA_SERVER} "pkill -f 'java -jar' || true"
                    ssh ${QA_SERVER} "${APP_START_COMMAND}"
                    """
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh """
                    scp target/*.jar ${APP_SERVER}:/home/ec2-user/app.jar
                    ssh ${APP_SERVER} "pkill -f 'java -jar' || true"
                    ssh ${APP_SERVER} "${APP_START_COMMAND}"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
