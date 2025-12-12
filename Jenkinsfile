pipeline {
    agent { label 'agent' }

    environment {
        MAVEN_OPTS = "-Dmaven.test.failure.ignore=false"
        APP_ARTIFACT = "target/*.jar"

        APP_USER = "ubuntu"
        APP_HOST = "10.30.1.200"
        APP_DEPLOY_DIR = "/opt/app"

        QA_USER = "ubuntu"
        QA_HOST = "10.30.1.201"
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
                sh '''
                  if ! command -v dependency-check.sh > /dev/null 2>&1; then
                    echo "WARNING: OWASP Dependency Check not installed"
                  else
                    dependency-check.sh --project "MyJavaApp" --scan . --format HTML --failOnCVSS 7
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

        stage('Deploy to QA (development branch)') {
            when {
                branch 'development'
            }
            steps {
                sshagent(credentials: ['applicationscreds']) {
                    sh """
                      scp ${APP_ARTIFACT} ${QA_USER}@${QA_HOST}:${QA_DEPLOY_DIR}/app.jar
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
                sshagent(credentials: ['applicationscreds']) {
                    sh """
                      scp ${APP_ARTIFACT} ${APP_USER}@${APP_HOST}:${APP_DEPLOY_DIR}/app.jar
                      ssh ${APP_USER}@${APP_HOST} 'pkill -f app.jar || true'
                      ssh ${APP_USER}@${APP_HOST} 'nohup java -jar ${APP_DEPLOY_DIR}/app.jar > app.log 2>&1 &'
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build succeeded!"
        }
        failure {
            echo "❌ Build failed!"
        }
    }
}
