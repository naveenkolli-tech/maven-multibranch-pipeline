pipeline {
    agent { label 'maven-agent' }

    environment {
        // Change these values as per your setup
        APP_NAME = "demo-app"
        ARTIFACT_DIR = "target"
        SSH_CREDENTIALS = "app-server-ssh"
        APP_SERVER_USER = "ec2-user"
        APP_SERVER_IP = "YOUR_APP_SERVER_IP"
        QA_SERVER_IP  = "YOUR_QA_SERVER_IP"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Checking out source code from GitHub"
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo "Running Maven clean test"
                sh 'mvn clean test'
            }
        }

        stage('Security Scan') {
            steps {
                echo "Running OWASP Dependency Check"
                sh '''
                  mvn org.owasp:dependency-check-maven:check \
                  -DfailBuildOnCVSS=7
                '''
            }
        }

        stage('Package') {
            steps {
                echo "Packaging application"
                sh 'mvn clean package -DskipTests'

                echo "Archiving artifact"
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy to QA (develop branch)') {
            when {
                branch 'develop'
            }
            steps {
                echo "Deploying to QA server"

                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    scp -o StrictHostKeyChecking=no target/*.jar \
                    ${APP_SERVER_USER}@${QA_SERVER_IP}:/home/${APP_SERVER_USER}/${APP_NAME}.jar

                    ssh ${APP_SERVER_USER}@${QA_SERVER_IP} '
                      pkill -f ${APP_NAME}.jar || true
                      nohup java -jar /home/${APP_SERVER_USER}/${APP_NAME}.jar > app.log 2>&1 &
                    '
                    """
                }
            }
        }

        stage('Deploy to Production (main branch)') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying to Production server"

                sshagent(credentials: [env.SSH_CREDENTIALS]) {
                    sh """
                    scp -o StrictHostKeyChecking=no target/*.jar \
                    ${APP_SERVER_USER}@${APP_SERVER_IP}:/home/${APP_SERVER_USER}/${APP_NAME}.jar

                    ssh ${APP_SERVER_USER}@${APP_SERVER_IP} '
                      pkill -f ${APP_NAME}.jar || true
                      nohup java -jar /home/${APP_SERVER_USER}/${APP_NAME}.jar > app.log 2>&1 &
                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check logs."
        }
    }
}
