pipeline {
    agent any

    environment {
        TEAMS_WEBHOOK_PROD = credentials('teams-webhook-prod')
        PROJECT = "cicd_poc"
        VERSION_PREFIX = "v1.0"
    }

    triggers {
        // Trigger only when a PR is merged into master or main
        GenericTrigger(
            genericVariables: [
                [key: 'action', value: '$.action'],
                [key: 'merged', value: '$.pull_request.merged'],
                [key: 'base_branch', value: '$.pull_request.base.ref'],
                [key: 'head_branch', value: '$.pull_request.head.ref']
            ],
            causeString: 'Triggered by PR ${action} (merged=${merged}) from ${head_branch} → ${base_branch}',
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
                    // Run only if PR is merged into master or main
                    return (
                        env.action == 'closed' &&
                        env.merged == 'true' &&
                        (env.base_branch == 'master' || env.base_branch == 'main')
                    )
                }
            }
            steps {
                echo "Pull Request merged into ${env.base_branch} — checking out latest code..."
                git branch: env.base_branch,
                    url: 'https://github.com/Surya-Jayachandran/mobitech1-cicd_poc.git'
            }
        }

        stage('Build Firmware') {
            when { expression { env.base_branch in ['master', 'main'] && env.merged == 'true' } }
            steps {
                echo "Building firmware..."
                // Add real build command here, for example:
                // sh 'make clean all'
            }
        }

        stage('Prepare Release Files') {
            when { expression { env.base_branch in ['master', 'main'] && env.merged == 'true' } }
            steps {
                script {
                    sh 'rm -rf release || true'
                    sh 'mkdir -p release'

                    def buildNum = env.BUILD_NUMBER
                    def version = VERSION_PREFIX
                    def binFile = "${PROJECT}_${version}_${buildNum}.bin"
                    def hexFile = "${PROJECT}_${version}_${buildNum}.hex"

                    // Try to find existing .bin and .hex files
                    def binSrc = sh(script: "find . -type f -name '*.bin' | head -n 1 || true", returnStdout: true).trim()
                    def hexSrc = sh(script: "find . -type f -name '*.hex' | head -n 1 || true", returnStdout: true).trim()

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
            when { expression { env.base_branch in ['master', 'main'] && env.merged == 'true' } }
            steps {
                script {
                    def buildNum = env.BUILD_NUMBER
                    def version = VERSION_PREFIX
                    def jobName = env.JOB_NAME
                    def fromBranch = env.head_branch ?: "unknown"
                    def toBranch = env.base_branch ?: "unknown"
                    def binFile = "${PROJECT}_${version}_${buildNum}.bin"
                    def hexFile = "${PROJECT}_${version}_${buildNum}.hex"
                    def buildUrl = env.BUILD_URL + "artifact/release/"

                    def releaseContent = fileExists('RELEASE.md')
                        ? readFile('RELEASE.md').trim()
                        : "No release notes found in RELEASE.md"

                    def message = """\
${jobName} Build Success<br>
──────────────────────────────<br>
Merged: ${fromBranch} → ${toBranch}<br>
Version: ${version}<br>
Build: ${buildNum}<br><br>
RELEASE.md:<br>
${releaseContent.replaceAll('\n', '<br>')}<br><br>
Files:<br>
<a href='${buildUrl}${binFile}'>${binFile}</a><br>
<a href='${buildUrl}${hexFile}'>${hexFile}</a>
"""

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
            when { expression { env.base_branch in ['master', 'main'] && env.merged == 'true' } }
            steps {
                echo "Archiving build artifacts..."
                archiveArtifacts artifacts: 'release/*.bin, release/*.hex',
                                  onlyIfSuccessful: true,
                                  fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Build completed successfully for PR merged into ${env.base_branch}."
        }

        failure {
            script {
                def failMsg = """\
${env.JOB_NAME} Build Failed<br>
──────────────────────────────<br>
Version: ${VERSION_PREFIX}<br>
Build: ${env.BUILD_NUMBER}<br>
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
