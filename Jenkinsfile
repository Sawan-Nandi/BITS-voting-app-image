pipeline {
    agent any
    environment {
        PROJECT_ID = 'devops-project-1-434509'
        GCR_REGION = 'gcr.io'
        SERVICE_ACCOUNT = credentials('bits-jenkins-sa-key')
        GITHUB_REPO = 'Sawan-Nandi/BITS-voting-app-image'
        GITHUB_CREDENTIALS = credentials('github-jenkins-pat')
        GITHUB_TOKEN = credentials('github-jenkins-pat')
    }
    triggers {
        githubPush() // This triggers the pipeline when changes are pushed to the repository
    }
    stages {
        stage('Cleanup and Checkout') {
            steps {
                // Cleanup any existing repository if already cloned
                script {
                    def repoDir = 'BITS-voting-app-image'
                    if (fileExists(repoDir)) {
                        echo "Removing existing repository directory: ${repoDir}"
                        sh "rm -rf ${repoDir}" // Remove the existing cloned repo
                    }
                    
                    // Now clone the repository
                    withCredentials([string(credentialsId: 'github-jenkins-pat', variable: 'GITHUB_TOKEN')]) {
                        sh """
                            git clone https://github.com/${GITHUB_REPO}.git
                            cd ${repoDir}
                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"
                        """
                    }
                }
            }
        }

        stage('Detect Changes and Build Images') {
            steps {
                script {
                    // Get the list of changed files
                    def changedFiles = sh(script: "git diff --name-only HEAD HEAD~1", returnStdout: true).trim().split("\n")
                    
                    // Define each microservice's directory and image name
                    def services = [
                        'vote': "${GCR_REGION}/${PROJECT_ID}/vote",
                        'worker': "${GCR_REGION}/${PROJECT_ID}/worker",
                        'result': "${GCR_REGION}/${PROJECT_ID}/result"
                    ]
                    
                    // Loop through each microservice directory
                    services.each { serviceDir, imageName ->
                        // Check if the Containerfile in the current service directory is changed
                        def shouldBuild = changedFiles.any { it.startsWith(serviceDir + "/") }
                        
                        if (shouldBuild) {
                            echo "Changes detected in ${serviceDir}. Building and pushing ${imageName}."

                            // Get the latest image tag for the current service from GCR
                            sh "gcloud auth activate-service-account --key-file=${SERVICE_ACCOUNT}"
                            def lastTag = sh(script: "gcloud container images list-tags ${imageName} --limit=1 --format='value(tags)'", returnStdout: true).trim()
                            def newTag = 'v1'

                            if (lastTag) {
                                // Increment the version by 1
                                def version = lastTag.replaceAll("[^0-9]", "")
                                newTag = "v${version.toInteger() + 1}"
                            }

                            echo "New version tag: ${newTag}"

                            // Build the container image for the modified microservice
                            sh """
                                gcloud auth activate-service-account --key-file=${SERVICE_ACCOUNT}
                                docker build -t ${imageName}:${newTag} -f ${serviceDir}/Containerfile ${serviceDir}
                            """
                            
                            // Push the image to GCR
                            sh """
                                gcloud auth configure-docker ${GCR_REGION}
                                docker push ${imageName}:${newTag}
                            """
                        } else {
                            echo "No changes detected in ${serviceDir}. Skipping build."
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline success. Check logs for details."
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}
