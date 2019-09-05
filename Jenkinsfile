#!groovy

// Copyright 2017 Intel Corporation
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
// ------------------------------------------------------------------------------

pipeline {
    agent {
        node {
            label 'master'
            customWorkspace "workspace/${env.BUILD_TAG}"
        }
    }

    triggers {
        cron(env.BRANCH_NAME == 'master' ? 'H 3 * * *' : '')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr: '31'))
    }

    environment {
        ISOLATION_ID = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
        COMPOSE_PROJECT_NAME = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
    }

    stages {
        stage('Check Whitelist') {
            steps {
                readTrusted 'bin/whitelist'
                sh './bin/whitelist "$CHANGE_AUTHOR" /etc/jenkins-authorized-builders'
            }
            when {
                not {
                    branch 'master'
                }
            }
        }

        stage('Check for Signed-Off Commits') {
            steps {
                sh '''#!/bin/bash -l
                    if [ -v CHANGE_URL ] ;
                    then
                        temp_url="$(echo $CHANGE_URL |sed s#github.com/#api.github.com/repos/#)/commits"
                        pull_url="$(echo $temp_url |sed s#pull#pulls#)"
                        IFS=$'\n'
                        for m in $(curl -s "$pull_url" | grep "message") ; do
                            if echo "$m" | grep -qi signed-off-by:
                            then
                              continue
                            else
                              echo "FAIL: Missing Signed-Off Field"
                              echo "$m"
                              exit 1
                            fi
                        done
                        unset IFS;
                    fi
                '''
            }
        }

        stage('Build Lint Requirements') {
            steps{
                sh 'docker-compose -f docker/compose/run-lint.yaml build'
                sh 'docker-compose -f docker/compose/sawtooth-build.yaml up'
                sh 'docker-compose -f docker/compose/sawtooth-build.yaml down'
            }
        }

        stage('Run Lint') {
            steps {
                sh 'docker-compose -f docker/compose/run-lint.yaml up --abort-on-container-exit --exit-code-from lint-rust lint-rust'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'INSTALL_TYPE="" ./bin/run_tests'
            }
        }

        stage('Build Documentation') {
            steps {
                sh 'docker build . -f docs/Dockerfile -t sawtooth-sdk-rust-docs:$ISOLATION_ID'
                sh 'docker run --rm -v $(pwd):/project/sawtooth-sdk-rust sawtooth-sdk-rust-docs:$ISOLATION_ID'
            }
        }

        stage('Build/archive artifacts') {
            steps {
                sh 'docker-compose -f docker-compose-installed.yaml build'
                sh 'docker-compose -f docker/compose/copy-debs.yaml up'
            }
        }
    }

    post {
        always {
            sh 'docker-compose -f docker/compose/sawtooth-build.yaml down'
            sh 'docker-compose -f docker/compose/run-lint.yaml down'
            sh 'docker-compose -f docker/compose/copy-debs.yaml down'
        }
        success {
            archiveArtifacts 'build/debs/*.deb, build/scar/*.scar, docs/build/**'
        }
        aborted {
            error "Aborted, exiting now"
        }
        failure {
            error "Failed, exiting now"
        }
    }
}
