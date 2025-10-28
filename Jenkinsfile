pipeline {
    agent any

    environment {
        TEAMS_WEBHOOK_PROD = credentials('teams-webhook-prod')
        PROJECT = "cicd_poc"
        VERSION_PREFIX = "v1.0"
    }

    triggers {
        // Trigger only when a pull request is merged into master
        GenericTrigger(
            genericVariables: [
                [key: 'action', value: '$.action'],
                [key: 'merged', value: '$.pull_request.merged'],
                [key: 'base_branch', value: '$.pull_request.base.ref'],
                [key: 'head_branch', value: '$.pull_request.head.ref']
            ],
            causeString: 'Triggered by Pull Request $action from $head_branch to $base_branch',
            token: 'github-pr-token',
            printContributedVariables: true,
            printPostContent: true,
            regexpFilterExpression: '^(closed)$',
            regexpFilterText: '$action'
        )
    }

    stages {

        stage('Checkout') {
            when {
                expression {
                    // Run only if PR is merged into master
                    return (env.action == 'closed' &&
                            env.merged == 'true' &&
                            env.base_branch == 'master')
                }
            }
            steps {
                echo "Pull Request merged into master — checking out latest code..."
                git branch: 'master',
                    url: 'https://github.com/Surya-Jayachandran/mobitech1-cicd_poc.git'
            }
        }

        stage('Build Firmware') {
            when { expression { env.base_branch == 'master' } }
            steps {
                echo "Building firmware..."
                // Add actual build command here if available
            }
        }

        stage('Prepare Release Files') {
            when { expression { env.base_branch == 'master' } }
            steps {
                script {
                    sh 'rm -rf release || true'
                    sh 'mkdir -p release'

                    def buildNum = env.BUILD_NUMBER
                    def version = VERSION_PREFIX
                    def binFile = "${PROJECT}_${version}_${buildNum}.bin"
                    def hexFile = "${PROJECT}_${version}_${buildNum}.hex"

                    // Detect and copy firmware files
                    def binSrc = sh(script: "find . -type f -name '*.bin' | grep -E '(KEIL|IAR|GCC|build|output)' | head -n 1 || true", returnStdout: true).trim()
                    def hexSrc = sh(script: "find . -type f -name '*.hex' | grep -E '(KEIL|IAR|GCC|build|output)' | head -n 1 || true", returnStdout: true).trim()

                    if (binSrc) {
                        sh "cp '${binSrc}' release/${binFile}"
                    } else {
                        sh "echo 'Binary placeholder' > release/${binFile}"
                    }

                    if (hexSrc) {
                        sh "cp '${hexSrc}' release/${hexFile}"
                    } else {
                        sh "echo 'Hex placeholder' > release/${hexFile}"
                    }

                    echo "Firmware prepared: ${binFile}, ${hexFile}"
                }
            }
        }

        stage('Notify Teams') {
            when { expression { env.base_branch == 'master' } }
            steps {
                script {
                    def buildNum = env.BUILD_NUMBER
                    def version = VERSION_PREFIX
                    def jobName = env.JOB_NAME
                    def triggeredBy = currentBuild.getBuildCauses()[0]?.userName ?: "Automated Trigger"
                    def binFile = "${PROJECT}_${version}_${buildNum}.bin"
                    def hexFile = "${PROJECT}_${version}_${buildNum}.hex"

                    def releaseContent = fileExists('RELEASE.md') ?
                        readFile('RELEASE.md').trim() :
                        "No release notes found in RELEASE.md"

                    def buildUrl = env.BUILD_URL + "artifact/release/"

                    def message = """\
${jobName} Build Success<br>
──────────────────────────────<br>
Version: ${version}<br>
Build: ${buildNum}<br>
Triggered By: ${triggeredBy}<br><br>
RELEASE.md:<br>
${releaseContent.replaceAll('\n', '<br>')}<br><br>
Files:<br>
<a href='${buildUrl}${binFile}'>${binFile}</a><br>
<a href='${buildUrl}${hexFile}'>${hexFile}</a>
"""

                    echo "Sending notification to Microsoft Teams..."
                    sh """
                        curl -H 'Content-Type: application/json' \
                             -d '{"text": "${message.replaceAll('"', '\\"')}"}' \
                             ${TEAMS_WEBHOOK_PROD}
                    """
                }
            }
        }

        stage('Archive Artifacts') {
            when { expression { env.base_branch == 'master' } }
            steps {
                echo "Archiving firmware artifacts..."
                archiveArtifacts artifacts: 'release/*.bin, release/*.hex, release/BuildInfo.txt',
                                  onlyIfSuccessful: true,
                                  fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Build completed successfully for PR merged into master."
        }

        failure {
            script {
                def failMsg = """\
${env.JOB_NAME} Build Failed<br>
──────────────────────────────<br>
Version: ${VERSION_PREFIX}<br>
Build: ${env.BUILD_NUMBER}<br>
Triggered By: ${currentBuild.getBuildCauses()[0]?.userName ?: "Automated Trigger"}<br>
"""
                echo "Sending failure message to Teams..."
                sh """
                    curl -H 'Content-Type: application/json' \
                         -d '{"text": "${failMsg.replaceAll('"', '\\"')}"}' \
                         ${TEAMS_WEBHOOK_PROD}
                """
            }
        }
    }
}
