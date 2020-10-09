@Library("jenkins_library") _

pipeline {
    agent {
        dockerfile {
            filename 'tests/ci/Dockerfile.build'
            args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    options {
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    tagName = sh(returnStdout: true, script: "git tag --points-at HEAD").trim()
                    isRelease = !tagName.isEmpty()
                    IMAGE_TAG = env.JOB_NAME + "." + env.BUILD_NUMBER
                    IMAGE_TAG = IMAGE_TAG.toLowerCase()
                    imageName = "blazemeter/taurus"
                    date = sh(returnStdout: true, script: "echo \$(date '+%Y-%m-%d')").trim()
                    commitSha = GIT_COMMIT.take(8)
                    imageTag = "${imageName}:master-${commitSha}-${date}"
                    extraImageTag = isRelease ? "${imageName}:${tagName} -t ${imageTag} -t ${imageName}:latest" : "${imageName}:unstable -t ${imageTag}"
                    sh "./build-info.sh ${isRelease}"
                }
            }
        }
        stage("Create artifacts") {
            steps {
                script {
                    sh "./build-artifacts.sh"
                }
                archiveArtifacts artifacts: 'dist/*.whl', fingerprint: true
                archiveArtifacts artifacts: 'build/nsis/*_x64.exe', fingerprint: true
            }
        }
        stage("Docker Image Build") {
            steps {
                script {
                    sh "docker build --no-cache -t ${JOB_NAME} -t ${extraImageTag} ."
                }
            }
        }
        stage("Integration Tests") {
            steps {
                sh """
                   docker run -v `pwd`:/bzt-configs -v `pwd`/integr-artifacts:/tmp/artifacts ${JOB_NAME} -sequential examples/all-executors.yml
                   """
            }
        }
        stage("Docker Image Push") {
            steps {
                withDockerRegistry([ credentialsId: "dockerhub-access", url: "" ]) {
                    sh "docker push ${imageName}"
                }
            }
        }
        stage("Deploy site") {
            steps {
                script {
                    PROJECT_ID = "blazemeter-taurus-website-prod"
                    withCredentials([file(credentialsId: PROJECT_ID, variable: 'CRED_JSON')]) {
                        sh """
                           gcloud auth activate-service-account --key-file ${CRED_JSON}
                           gcloud config set project ${PROJECT_ID}
                           gcloud config set compute/zone us-central1-a
                           """
                    }
                    sh """
                       export PROJECT_ID=${PROJECT_ID}
                       ./site/deploy-site.sh ${isRelease}
                       """
                }
            }
        }
    }
    post {
        always {
            smartSlackNotification(channel: "taurus-dev", buildStatus:currentBuild.result ?: 'SUCCESS')
            cleanWs()
        }
    }
}
