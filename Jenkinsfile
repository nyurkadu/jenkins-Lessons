// Declarative Pipeline for the "git and pipeline" job.
// Jenkins pulls this file from GitHub on every build (Pipeline script from SCM),
// bumps the patch version in version.txt, pushes it back and deploys
// to Staging (even version) or Production (odd version).
pipeline {
    // Run on any available agent (here: the built-in node inside the Jenkins container).
    agent any

    // Global variables available in every stage as ${GIT_REPO} / env.GIT_REPO.
    environment {
        GIT_REPO   = 'github.com/nyurkadu/jenkins-Lessons'   // repo without https:// so the token can be prepended
        GIT_BRANCH = 'main'                                   // branch the new version is pushed to
    }

    stages {
        // 1. Show what Jenkins checked out into the workspace.
        stage('Checkout & Verification') {
            steps {
                echo 'Checking workspace directory...'
                sh 'ls -la'
            }
        }

        // 2. Placeholder for real tests (unit tests, linters, etc.).
        stage('Test App') {
            steps {
                echo 'Simulating test execution on code pulled from Git...'
                sh 'echo "Code tests passed successfully!"'
            }
        }

        // 3. Print the version that is currently committed in Git.
        stage('Read Version') {
            steps {
                echo 'Reading version.txt...'
                sh 'cat version.txt'
            }
        }

        // 4. Increase the last (patch) number by 1: v1.0.1 -> v1.0.2, v1.0.9 -> v1.0.10.
        stage('Bump Version') {
            steps {
                echo 'Incrementing patch version by 1...'
                // awk splits the version by "." (-F.), adds 1 to the last field ($NF)
                // and joins the fields back with "." (OFS=.).
                sh '''
                    OLD_VERSION=$(cat version.txt)
                    NEW_VERSION=$(echo "$OLD_VERSION" | awk -F. -v OFS=. '{ $NF = $NF + 1; print }')
                    echo "$NEW_VERSION" > version.txt
                    echo "Version bumped: $OLD_VERSION -> $NEW_VERSION"
                '''
                // Export the new version to env so that the "when" conditions below can use it.
                script {
                    env.NEW_VERSION  = readFile('version.txt').trim()          // e.g. v1.0.2
                    env.PATCH_NUMBER = env.NEW_VERSION.tokenize('.').last()    // e.g. 2
                    echo "Patch number ${env.PATCH_NUMBER} is ${env.PATCH_NUMBER.toInteger() % 2 == 0 ? 'even -> Staging' : 'odd -> Production'}"
                }
            }
        }

        // 5. Commit the bumped version.txt and push it back to GitHub,
        //    so the next build starts from the new version.
        stage('Push Version') {
            steps {
                echo 'Committing new version back to GitHub...'
                // "github-token" is a Username/Password credential stored encrypted in Jenkins.
                // withCredentials exposes it as GIT_USER / GIT_TOKEN and masks the token in the log.
                withCredentials([usernamePassword(credentialsId: 'github-token',
                                                  usernameVariable: 'GIT_USER',
                                                  passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        NEW_VERSION=$(cat version.txt)
                        git config user.email "jenkins@localhost"
                        git config user.name "Jenkins CI"
                        git add version.txt
                        # [ci skip] marks the commit as made by CI (useful if SCM polling/webhooks are enabled later)
                        git commit -m "Bump version to $NEW_VERSION [ci skip]"
                        # The workspace is in detached HEAD state, so push the current commit to the target branch
                        git push "https://${GIT_USER}:${GIT_TOKEN}@${GIT_REPO}" HEAD:${GIT_BRANCH}
                    '''
                }
            }
        }

        // 6a. Even patch number (v1.0.2, v1.0.4, ...) -> Staging.
        //     "when" skips the stage (shown grey in Stage View) if the condition is false.
        stage('Deploy to Staging') {
            when {
                expression { env.PATCH_NUMBER.toInteger() % 2 == 0 }
            }
            steps {
                echo 'Even version -> deploying application to Staging...'
                sh 'echo "App $(cat version.txt) successfully deployed to Staging!"'
            }
        }

        // 6b. Odd patch number (v1.0.1, v1.0.3, ...) -> Production.
        stage('Deploy to Production') {
            when {
                expression { env.PATCH_NUMBER.toInteger() % 2 != 0 }
            }
            steps {
                echo 'Odd version -> deploying application to Production...'
                sh 'echo "App $(cat version.txt) successfully deployed to Production!"'
            }
        }
    }
}
