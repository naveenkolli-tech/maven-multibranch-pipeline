pipeline {
    agent { label 'maven-agent' }

    environment {
        // Change to your repo and artifact details
        MAVEN_OPTS = "-Dmaven.test.failure.ignore=false"
        APP_ARTIFACT = "target/app.jar"      // adjust to your built jar/war
        // Application server details
        APP_USER = "ubuntu"
        APP_HOST = "APP_SERVER_PUBLIC_IP"    // e.g. 3.x.x.x
        APP_DEPLOY_DIR = "/opt/app"
        // QA server (for develop branch, extra)
        QA_USER = "ubuntu"
        QA_HOST = "QA_SERVER_PUBLIC_IP"
        QA_DEPLOY_DIR = "/opt/app-qa"
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
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Security Scan') {
            steps {
                // Example using OWASP Dependency-Check CLI if installed on agent
                // Adjust the path and project name as needed
                sh '''
                    if ! command -v dependency-check.sh > /dev/null 2>&1; then
                      echo "WARNING: dependency-check.sh not installed, skipping scan"
                    else
                      dependency-check.sh \
                        --project "MyJavaApp" \
                        --scan . \
                        --format "HTML" \
                        --out dependency-check-report.html \
                        --failOnCVSS 7
                    fi
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B package -DskipTests'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy to QA (develop branch)') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh """
                      scp target/*.jar ${QA_USER}@${QA_HOST}:${QA_DEPLOY_DIR}/app.jar
                      ssh ${QA_USER}@${QA_HOST} 'pkill -f app.jar || true'
                      ssh ${QA_USER}@${QA_HOST} 'nohup java -jar ${QA_DEPLOY_DIR}/app.jar > app.log 2>&1 &'
                    """
                }
            }
        }

        stage('Deploy to Application Server (main branch)') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh """
                      scp target/*.jar ${APP_USER}@${APP_HOST}:${APP_DEPLOY_DIR}/app.jar
                      ssh ${APP_USER}@${APP_HOST} 'pkill -f app.jar || true'
                      ssh ${APP_USER}@${APP_HOST} 'nohup java -jar ${APP_DEPLOY_DIR}/app.jar > app.log 2>&1 &'
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "Build failed!"
        }
        success {
            echo "Build succeeded!"
        }
    }
}
