pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Build') {
            steps {
                echo "Building Maven project"
            }
        }
    }
}
