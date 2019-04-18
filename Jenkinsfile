#!/usr/bin/env groovy

def projectName = "sp-demo-web"
def imageName = "alex202/sp-demo-web"

def gitCommit = null
def gitBranch = null
def imageTag = null
def buildDate = null

def version = "0.1"

podTemplate(label: 'mypod', containers: [
    containerTemplate(name: 'golang', image: 'golang:1.8', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: 'docker', ttyEnabled: true, command: 'cat')
  ],
  envVars: [

  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]) {

    node('mypod') {

        checkout scm

        // print environment variables
        echo sh(script: 'env|sort', returnStdout: true)

        sh "git rev-parse --short HEAD > .git/commit-id"
        gitCommit = readFile('.git/commit-id').trim()

        // git branch name is taken from an env var for multi-branch pipeline project, or from git for other projects
        if (env['BRANCH_NAME']) {
            gitBranch = BRANCH_NAME
        } else {
            sh "git rev-parse --abbrev-ref HEAD > .git/branch-name"
            gitBranch = readFile('.git/branch-name').trim()

        }

        imageTag = "${gitBranch}-${gitCommit}"

        sh "date +'%Y-%m-%d %H-%M-%S' > .git/build-date"
        buildDate = readFile('.git/build-date').trim()


        def buildInfo = """# Build info
BUILD_NUMBER=${env.BUILD_NUMBER}
BUILD_DATE=${buildDate}
BUILD_GIT_COMMIT=${gitCommit}
BUILD_GIT_BRANCH=${gitBranch}
DOCKER_IMAGE_TAG=${version}.${env.BUILD_NUMBER}
"""

        echo buildInfo

        stage('Build go binaries') {
            container('golang') {

                def pwd = pwd()

                sh """
                    mkdir -p /go/src/github.com
                    ln -s $pwd /go/src/github.com/${projectName}
                    ls -la /go/src/github.com
                    cd /go/src/github.com/${projectName}
                    go get && GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o target/demo-webserver
                """
            }
        }

        stage('Build and push docker image') {
            container('docker') {

                withCredentials([[$class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_HUB_USER',
                        passwordVariable: 'DOCKER_HUB_PASSWORD']]) {

                    sh """
                      docker build --force-rm \
                            --build-arg BUILD_DATE="${buildDate}" \
                            --build-arg IMAGE_TAG_REF=${imageTag} \
                            --build-arg VCS_REF=${gitCommit} \
                            -t ${imageName}:${imageTag} .
                      """
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD} "
                    sh "docker push ${imageName}:${imageTag} "
                }
            }
        }
    }
}