pipeline {
    agent any

    environment {
        TEAMS_WEBHOOK_PROD = credentials('teams-webhook-prod')
        PROJECT = "cicd_poc"
        VERSION_PREFIX = "v1.0"
    }

    triggers {
        githubPush() // automatically trigger from GitHub webhook
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source from GitHub..."
                git branch: 'release/1.12',
                    url: 'https://github.com/Surya-Jayachandran/mobitech1-cicd_poc.git'
            }
        }

        stage('Build Firmware') {
            steps {
                echo "Building firmware..."
                // Replace this with your actual build command if applicable
                // Example for Keil:
                // bat '"C:\\Keil_v5\\UV4\\UV4.exe" -b KEIL\\cicd_poc.uvprojx -j0 -o KEIL\\output\\build.log'
                // Example for GCC:
                // sh 'make clean all'
            }
        }

        stage('Prepare Release Files') {
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
                        echo "Found binary: ${binSrc}"
                        sh "cp '${binSrc}' release/${binFile}"
                    } else {
                        echo "No .bin found — creating placeholder"
                        sh "echo 'Binary placeholder' > release/${binFile}"
                    }

                    if (hexSrc) {
                        echo "Found hex: ${hexSrc}"
                        sh "cp '${hexSrc}' release/${hexFile}"
                    } else {
                        echo "No .hex found — creating placeholder"
                        sh "echo 'Hex placeholder' > release/${hexFile}"
                    }

                    echo "Firmware prepared: ${binFile}, ${hexFile}"
                }
            }
        }

        stage('Notify Teams') {
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
                    def binUrl = "${buildUrl}${binFile}"
                    def hexUrl = "${buildUrl}${hexFile}"

                    def message = """\
${jobName} Build Success<br>
──────────────────────────────<br>
Version: ${version}<br>
Build: ${buildNum}<br>
Triggered By: ${triggeredBy}<br><br>
Release Notes:<br>
${releaseContent.replaceAll('\n', '<br>')}<br><br>
Files:<br>
<a href='${binUrl}'>${binFile}</a><br>
<a href='${hexUrl}'>${hexFile}</a>
"""

                    // Save Build Info for reference
                    writeFile file: 'release/BuildInfo.txt', text: message.replaceAll('<br>', '\n')

                    echo "Sending build success message to Microsoft Teams..."
                    sh """
                        curl -H 'Content-Type: application/json' \
                             -d '{"text": "${message.replaceAll('"', '\\"')}"}' \
                             ${TEAMS_WEBHOOK_PROD}
                    """
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo "Archiving build outputs..."
                archiveArtifacts artifacts: 'release/*.bin, release/*.hex, release/BuildInfo.txt',
                                  onlyIfSuccessful: true,
                                  fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Build completed successfully for ${env.JOB_NAME}"
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
                echo "Sending failure message to Microsoft Teams..."
                sh """
                    curl -H 'Content-Type: application/json' \
                         -d '{"text": "${failMsg.replaceAll('"', '\\"')}"}' \
                         ${TEAMS_WEBHOOK_PROD}
                """
            }
        }
    }
}
