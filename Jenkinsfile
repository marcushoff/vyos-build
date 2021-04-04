#!/usr/bin/env groovy
// Copyright (C) 2019-2021 VyOS maintainers and contributors
//
// This program is free software; you can redistribute it and/or modify
// in order to easy exprort images built to "external" world
// it under the terms of the GNU General Public License version 2 or later as
// published by the Free Software Foundation.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License
// along with this program.  If not, see <http://www.gnu.org/licenses/>.
@NonCPS

// Using a version specifier library, use 'current' branch. The underscore (_)
// is not a typo! You need this underscore if the line immediately after the
// @Library annotation is not an import statement!
@Library('vyos-build@current')_
setDescription()

// Due to long build times on DockerHub we rather build the container by ourself
// and publish it later on.

// create container names on demand
env.DOCKER_IMAGE =       "vyos/vyos-build:" + getGitBranchName()
env.DOCKER_IMAGE_ARM =   "vyos/vyos-build:" + getGitBranchName() + "-armhf"
env.DOCKER_IMAGE_ARM64 = "vyos/vyos-build:" + getGitBranchName() + "-arm64"

node('Docker') {
    stage('Fetch') {
        git branch: getGitBranchName(),
            url: getGitRepoURL()
    }
    stage('Build Docker container') {
        parallel (
            'x86-64': {
                script {
                    dir('docker') {
                        sh """
                            docker build -t ${env.DOCKER_IMAGE} .
                        """
                        if (! isCustomBuild()) {
                            withDockerRegistry([credentialsId: "DockerHub"]) {
                                sh "docker push ${env.DOCKER_IMAGE}"
                            }

                        }
                    }
                }
            },
//          'armhf': {
//              script {
//                  dir('docker') {
//                      sh """
//                          cp Dockerfile armhf/Dockerfile
//                          cp entrypoint.sh armhf/entrypoint.sh
//                          sed -i 's#^FROM.*#FROM multiarch/debian-debootstrap:armhf-buster-slim#' armhf/Dockerfile
//                          docker build -t ${env.DOCKER_IMAGE_ARM} armhf
//                      """
//                      if (! isCustomBuild()) {
//                          withDockerRegistry([credentialsId: "DockerHub"]) {
//                              sh "docker push ${env.DOCKER_IMAGE_ARM}"
//                          }
//                      }
//                  }
//              }
//          },
            'arm64': {
                script {
                    dir('docker') {
                        sh """
                            docker build -t ${env.DOCKER_IMAGE_ARM64} --build-arg ARCH=arm64v8/ .

                        """

                        if (! isCustomBuild()) {
                            withDockerRegistry([credentialsId: "DockerHub"]) {
                                sh "docker push ${env.DOCKER_IMAGE_ARM64}"
                            }
                        }
                    }
                }
            }
        )
    }
    stage('Build timestamp') {
        script {
            env.TIMESTAMP = sh(returnStdout: true, script: 'date +%Y%m%d%H%M').toString().trim()
        }
    }
}

pipeline {
    options {
        disableConcurrentBuilds()
        timeout(time: 150, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    parameters {
        string(name: 'BUILD_BY', defaultValue: 'autobuild@vyos.net', description: 'Builder identifier (e.g. jrandomhacker@example.net)')
        string(name: 'BUILD_VERSION', defaultValue: '1.4-rolling-' + env.TIMESTAMP, description: 'Version number (release builds only)')
        booleanParam(name: 'BUILD_PUBLISH', defaultValue: true, description: 'Publish this build to downloads.vyos.io and AWS S3')
        booleanParam(name: 'BUILD_SMOKETESTS', defaultValue: true, description: 'Include Smoketests in ISO image')
        booleanParam(name: 'BUILD_SNAPSHOT', defaultValue: false, description: 'Upload image to AWS S3 snapshot bucket')
        booleanParam(name: 'BUILD_QEMU', defaultValue: false, description: 'Generate QEMU image')
    }
    triggers {
        cron('H 2 * * *')
    }
    agent {
        dockerfile {
            reuseNode true
            filename 'Dockerfile'
            dir 'docker'
            args '--privileged --sysctl net.ipv6.conf.lo.disable_ipv6=0 -e GOSU_UID=1006 -e GOSU_GID=1006'
        }
    }
    stages {
        stage('Build ISO') {
            when {
                beforeOptions true
                beforeAgent true
                // Do not run ISO build when the Docker container definition or the build pipeline
                // library changes as this has no direct impact on the ISO image.
                not { changeset "**/docker/*" }
                not { changeset "**/vars/*" }
                not { changeset "**/packages/*" }
                anyOf {
                    triggeredBy 'TimerTrigger'
                    triggeredBy cause: "UserIdCause"
                }
            }
            steps {
                script {
                    // Display Git commit Id used with the Jenkinsfile on the Job "Build History" pane
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short=11 HEAD').trim()
                    currentBuild.description = sprintf('Git SHA1: %s', commitId[-11..-1])

                    def CUSTOM_PACKAGES = ''
                    if (params.BUILD_SMOKETESTS)
                        CUSTOM_PACKAGES = '--custom-package vyos-1x-smoketest'

                    sh """
                        ./configure \
                            --build-by ${params.BUILD_BY} \
                            --debian-mirror http://ftp.us.debian.org/debian/ \
                            --build-type release \
                            --version ${params.BUILD_VERSION} ${CUSTOM_PACKAGES}
                        sudo make iso
                    """

                    if (fileExists('build/live-image-amd64.hybrid.iso') == false) {
                        error('ISO build error')
                    }
                }
            }
        }
        stage('Test') {
            when {
                expression { return params.BUILD_SMOKETESTS }
            }
            parallel {
                stage('Smoketests') {
                    when {
                        expression { fileExists 'build/live-image-amd64.hybrid.iso' }
                    }
                    steps {
                        sh "sudo make test"
                    }
                }
                stage('Smoketests with vyos-configd and arbitrary config loader') {
                    when {
                        expression { fileExists 'build/live-image-amd64.hybrid.iso' }
                    }
                    steps {
                        sh "sudo make testc"
                    }
                }
            }
        }
        stage('Build QEMU image') {
            when {
                expression { fileExists 'build/live-image-amd64.hybrid.iso' }
                expression { return params.BUILD_QEMU }
            }
            steps {
                sh "sudo make qemu"
            }
        }
    }
    post {
        success {
            script {
                // only deploy ISO if build from official repository
                if (isCustomBuild())
                    return

                files = findFiles(glob: 'build/vyos*.iso')
                // Publish ISO image to daily builds bucket
                if (files && params.BUILD_PUBLISH) {
                    // Publish build result, using SSH-dev.packages.vyos.net Jenkins Credentials
                    sshagent(['SSH-dev.packages.vyos.net']) {
                        dir('build') {
                            // build up some fancy groovy variables so we do not need to write/copy
                            // every option over and over again!
                            def ARCH = sh(returnStdout: true, script: "dpkg --print-architecture").trim()
                            def ISO = sh(returnStdout: true, script: "ls vyos-*.iso").trim()
                            def SSH_DIR = '/home/sentrium/web/downloads.vyos.io/public_html/rolling/' + getGitBranchName() + '/' + ARCH
                            def SSH_OPTS = '-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
                            def SSH_REMOTE = env.DOWNLOADS_VYOS_IO_HOST // defined as global variable

                            // No need to explicitly check the return code. The pipeline
                            // will fail if sh returns a non 0 exit code
                            sh """
                                sha256sum ${ISO} > ${ISO}.sha256
                                ssh ${SSH_OPTS} ${SSH_REMOTE} -t "bash --login -c 'mkdir -p ${SSH_DIR}'"
                                ssh ${SSH_OPTS} ${SSH_REMOTE} -t "bash --login -c 'find ${SSH_DIR} -type f -mtime +14 -exec rm -f {} \\;'"
                                scp ${SSH_OPTS} -r ${ISO} ${ISO}.sha256 ${SSH_REMOTE}:${SSH_DIR}/
                                ssh ${SSH_OPTS} ${SSH_REMOTE} -t "bash --login -c '/usr/bin/make-latest-rolling-symlink.sh'"
                            """
                        }
                    }
                    withAWS(region: 'us-east-1', credentials: 's3-vyos-downloads-rolling-rw') {
                        s3Upload(bucket: 's3-us.vyos.io', path: 'rolling/',
                                 workingDir: 'build', includePathPattern: 'vyos*.iso')
                        s3Copy(fromBucket: 's3-us.vyos.io', fromPath: getGitBranchName() + '/' + files[0].name,
                               toBucket: 's3-us.vyos.io', toPath: getGitBranchName() + '/vyos-rolling-latest.iso')
                    }
                }

                // Publish ISO image to snapshot bucket
                if (files && params.BUILD_SNAPSHOT) {
                    withAWS(region: 'us-east-1', credentials: 's3-vyos-downloads-rolling-rw') {
                        s3Upload(bucket: 's3-us.vyos.io', path: 'snapshot/',
                                 workingDir: 'build', includePathPattern: 'vyos*.iso')
                    }
                }
            }
        }
        failure {
            archiveArtifacts artifacts: 'build/vyos*.iso, build/vyos*.qcow2',
                allowEmptyArchive: true
        }
        cleanup {
            echo 'One way or another, I have finished'
            // the 'build' directory got elevated permissions during the build
            // cdjust permissions so it can be cleaned up by the regular user
            sh 'sudo make purge'
            deleteDir() /* cleanup our workspace */
        }
    }
}
