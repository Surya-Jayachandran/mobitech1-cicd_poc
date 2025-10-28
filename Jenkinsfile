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
                    // Run only when PR is merged into master
                    return (env.action == 'closed' &&
                            env.merged == 'true' &&
                            env.base_branch == 'master')
                }
            }
            steps {
                echo "Pull Request merged into master — checking out code..."
                git branch: 'master',
                    url: 'https://github.com/Surya-Jayachandran/mobitech1-cicd_poc.git'
            }
        }

        stage('Build Firmware') {
            when { expression { env.base_branch == 'master' } }
            steps {
                echo "Building firmware..."
                // Add your actual build command here
                // Example:
                // sh 'make clean all'
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

                    // Try to find real firmware outputs automatically
                    def binSrc = sh(script: "find . -type f -name '*.bin' | grep -E '(KEIL|IAR|GCC|build|output)' | head -n 1 || true", returnStdout: true).trim()
                    def hexSrc = sh(script: "find . -type f -name '*.hex' | grep -E '(KEIL|IAR|GCC|build|output)' | head -n 1 || true", returnStdout: true).trim()

                    if (binSrc) {
                        echo "Found binary file: ${binSrc}"
                        sh "cp '${binSrc}' release/${binFile}"
                    } else {
                        echo "No .bin file found, creating placeholder"
                        sh "echo 'Binary placeholder' > release/${binFile}"
                    }

                    if (hexSrc) {
                        echo "Found hex file: ${hexSrc}"
                        sh "cp '${hexSrc}' release/${hexFile}"
                    } else {
                        echo "No .hex file found, creating placeholder"
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

                    def releaseContent = fileExists('RELEASE.md')
                        ? readFile('RELEASE.md').trim()
                        : "No release notes found in RELEASE.md"

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

                    // Save build info
                    writeFile file: 'release/BuildInfo.txt', text: message.replaceAll('<br>', '\n')

                    echo "Sending Teams notification..."
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
                echo "Archiving release files..."
                archiveArtifacts artifacts: 'release/*.bin, release/*.hex, release/BuildInfo.txt',
                                  onlyIfSuccessful: true,
                                  fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Build completed successfully for merged PR into master."
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
