pipeline {
    agent any

    stages {
        stage('Checkout & Verification') {
            steps {
                echo 'Checking workspace directory...'
                sh 'ls -la'
            }
        }
        stage('Test App') {
            steps {
                echo 'Simulating test execution on code pulled from Git...'
                sh 'echo "Code tests passed successfully!"'
            }
        }
        stage('Read Version') {
            steps {
                echo 'Reading version.txt...'
                sh 'cat version.txt'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh 'echo "App successfully deployed to Staging!"'
            }
        }
    }
}
