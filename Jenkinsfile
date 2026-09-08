pipeline {
    agent any

    environment {
        GIT_REPO   = 'github.com/nyurkadu/jenkins-Lessons'
        GIT_BRANCH = 'main'
    }

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
        stage('Bump Version') {
            steps {
                echo 'Incrementing patch version by 1...'
                sh '''
                    OLD_VERSION=$(cat version.txt)
                    NEW_VERSION=$(echo "$OLD_VERSION" | awk -F. -v OFS=. '{ $NF = $NF + 1; print }')
                    echo "$NEW_VERSION" > version.txt
                    echo "Version bumped: $OLD_VERSION -> $NEW_VERSION"
                '''
            }
        }
        stage('Push Version') {
            steps {
                echo 'Committing new version back to GitHub...'
                withCredentials([usernamePassword(credentialsId: 'github-token',
                                                  usernameVariable: 'GIT_USER',
                                                  passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        NEW_VERSION=$(cat version.txt)
                        git config user.email "jenkins@localhost"
                        git config user.name "Jenkins CI"
                        git add version.txt
                        git commit -m "Bump version to $NEW_VERSION [ci skip]"
                        git push "https://${GIT_USER}:${GIT_TOKEN}@${GIT_REPO}" HEAD:${GIT_BRANCH}
                    '''
                }
            }
        }
        stage('Deploy to Staging') {
            steps {
                echo 'Deploying application to Staging...'
                sh 'echo "App $(cat version.txt) successfully deployed to Staging!"'
            }
        }
        stage('Deploy to Production') {
            steps {
                echo 'Deploying application to Production...'
                sh 'echo "App $(cat version.txt) successfully deployed to Production!"'
            }
        }
    }
}
