pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'echo "Success!"; exit 0'
            }
        }
        stage('Test') {
            parallel {
                stage('Test On Windows') {
                    agent {
                        label "Windows"
                    }
                    steps {
                        echo 'This is a parallel test on Windows'
                    }
                }
                stage('Test On MacOS') {
                    agent {
                        label "MacOS"
                    }
                    steps {
                        echo 'is a parallel test on MacOS'
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'This will always run 007'
        }
        success {
            echo 'This will run only if successful'
        }
        failure {
            echo 'This will run only if failed'
        }
        unstable {
            echo 'This will run only if the run was marked as unstable'
        }
        changed {
            echo 'This will run only if the state of the Pipeline has changed'
            echo 'For example, if the Pipeline was previously failing but is now successful'
        }
    }
}