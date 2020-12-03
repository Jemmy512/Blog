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
                stage('Windows Test') {
                    agent {
                        label "Windows"
                    }
                    steps {
                        script {
                            try {
                                echo 'This is a parallel test on Windows'
                                setStatus(credentialsId: 'webex-pipeline', state: 'success', description: 'Windows Test')
                            } catch (Exception e) {
                                setStatus(credentialsId: 'webex-pipeline', state: 'failure', description: 'Windows Test')
                            }
                        }
                    }
                }
                stage('MacOS Test') {
                    agent {
                        label "MacOS"
                    }
                    steps {
                        script {
                            try {
                                echo 'This a parallel test on MacOS'
                                setStatus(credentialsId: 'webex-pipeline', state: 'success', description: 'MacOS Test')
                            } catch (Exception e) {
                                setStatus(credentialsId: 'webex-pipeline', state: 'success', description: 'MacOS Test')
                            }
                        }
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

def setStatus(args = [:]) {
    // withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: args.credentialsId , usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
    withCredentials([string(credentialsId: args.credentialsId, variable: 'GIT_PASSWORD')]) {
        try {
            def state
            if (args.state) {
                state = args.state
            } else if (currentBuild.currentResult == 'SUCCESS') {
                state = 'success'
            } else if (currentBuild.currentResult == 'UNSTABLE') {
                state = 'failure'
            } else if (currentBuild.currentResult == 'FAILURE') {
                state = 'error'
            }

            def targetUrl = args.targetUrl ?: "${env.BUILD_URL}display/redirect"
            // def context = args.context ?: 'continuous-integration/jenkins'
            def description = args.description
            def sha = args.commit ?: env.GIT_COMMIT_FULL

            echo "state: $state"
            echo "target_url: $targetUrl"
            echo "description: $description"

            def postPayload = JsonOutput.toJson([
                state: state,
                target_url: targetUrl,
                description: description
            ])

            def postUrl = (args.repoApiUrl ?: env.GITHUB_API_URL) + "statuses/$sha"
            httpRequest(url: postUrl,
                    requestBody: postPayload,
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    customHeaders: [[name: 'Authorization', value: "token $GIT_PASSWORD", maskValue: true]],
                    validResponseCodes: '200:299',
                    consoleLogResponseBody: true)
        } catch (e) {
            echo "$e"
        }

        // If prLabel is passed in, add that as a label to the PR
        if (args.prLabel) {
            try {
                def postUrl = (args.repoApiUrl ?: env.GITHUB_API_URL) + "issues/$CHANGE_ID/labels"

                httpRequest(url: postUrl,
                        requestBody: JsonOutput.toJson([labels: [args.prLabel]]),
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        customHeaders: [[name: 'Authorization', value: "token $GIT_PASSWORD", maskValue: true]],
                        validResponseCodes: '200:299',
                        consoleLogResponseBody: true)
            } catch (e) {
                echo "$e"
            }
        }
    }
}