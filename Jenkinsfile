pipeline {
    agent none
    options {
        skipStagesAfterUnstable()
        ansiColor('xterm')
        buildDiscarder(logRotator(artifactDaysToKeepStr: '20'))
    }
    parameters {
        string(name: 'MOD_PROMETHEUS_VERSION', defaultValue: '0.8', description: 'mod_prometheus version to install')
    }
    stages {
        stage('Build mod_prometheus') {
            agent {
                kubernetes {
                    defaultContainer 'rust'
                    yamlFile 'libon/jenkins/mod_prometheus-build-stage.yaml'
                    yamlMergeStrategy([$class: 'org.csanchez.jenkins.plugins.kubernetes.pod.yaml.Merge'])
                }
            }
            steps {
                script {
                    sh """
                        git clone https://github.com/libon/mod_prometheus.git

                        cd mod_prometheus && git checkout ${params.MOD_PROMETHEUS_VERSION}

                        cargo build

                        cp target/debug/libmod_prometheus.so ../mod_prometheus.so
                    """
                }
                stash(name: "mod_prometheus", includes: "mod_prometheus.so")
                milestone ordinal: 10, label: 'Building mod_prometheus'
            }
        }
        stage('Build freeswitch') {
            agent {
                kubernetes {
                    defaultContainer 'build'
                    yamlFile 'libon/jenkins/freeswitch-build-stage.yaml'
                    yamlMergeStrategy([$class: 'org.csanchez.jenkins.plugins.kubernetes.pod.yaml.Merge'])
                }
            }
            steps {
                script {
                    sh """

                        git config --global --add safe.directory `pwd`

                        if [ "${env.BRANCH_NAME}" = "master" ]; then
                            COMMIT=`git rev-parse HEAD`
                            FS_VERSION="master-\$COMMIT"
                        else
                            FS_VERSION=`git describe --tags`
                        fi
                        
                        echo -n \$FS_VERSION > version
                        
                        git config --global --add safe.directory `pwd`

                        ./bootstrap.sh -j

                        ./configure --prefix=/opt/freeswitch

                        make && make install && make sounds-install && make moh-install

                        tar -zcvf freeswitch.tar.gz /opt/freeswitch
                    """
                    env.FS_VERSION = sh(returnStdout: true, script: "cat version").trim()
                }
                stash(name: "freeswitch_bin", includes: "freeswitch.tar.gz")
                milestone ordinal: 20, label: 'Building FreeSWITCH'
            }
        }
        stage('Build/push docker image') {
            agent {
                kubernetes {
                    yamlFile 'libon/jenkins/image-build-stage.yaml'
                    yamlMergeStrategy([$class: 'org.csanchez.jenkins.plugins.kubernetes.pod.yaml.Merge'])
                }
            }
            steps {
                withCredentials([string(credentialsId: 'SIGNALWIRE_TOKEN', variable: 'SIGNALWIRE_TOKEN')]) {
                    unstash(name: "mod_prometheus")
                    unstash(name: "freeswitch_bin")
                    container(name:"kaniko", shell: '/busybox/sh') {
                        withEnv(['PATH+EXTRA=/busybox:/kaniko']) {
                            sh """#!/busybox/sh
                            /kaniko/executor --context `pwd` \
                                --build-arg SIGNALWIRE_TOKEN=$SIGNALWIRE_TOKEN \
                                --dockerfile=libon/docker/Dockerfile \
                                --destination=europe-west1-docker.pkg.dev/libon-build/images/freeswitch:${env.FS_VERSION}
                            """
                        }
                    }
                }
                milestone ordinal: 30, label: 'Building image'
            }
        }
        stage('Release') {
            when {
                branch 'v1.10'
            }

            steps {
                script {
                    releaseVersion = input(message: 'Should we release?', parameters: [
                        string(description: 'What is the next version', name: 'RELEASE_VERSION')
                    ])
                }

                podTemplate(
                        yamlMergeStrategy: merge(),
                        containers: [
                                containerTemplate(name: 'gcloud',
                                                  image: 'google/cloud-sdk:473.0.0-alpine',
                                                  command: 'cat',
                                                  ttyEnabled: true,
                                                  resourceRequestCpu: '100m',
                                                  resourceLimitMemory: '256Mi')
                        ]) {
                    node(POD_LABEL) {
                        ansiColor('xterm') {
                            script {
                                def scmVars = checkout scm
                                gitCommit = scmVars.GIT_COMMIT
                                try {
                                    milestone ordinal: 300, label: 'Release'
                                    githubTag(repository: "libon/freeswitch",
                                              commit: gitCommit,
                                              tag: releaseVersion,
                                              credentialsId: 'github-app')
                                    container('gcloud') {
                                        sh """
                                            gcloud artifacts docker tags add europe-west1-docker.pkg.dev/libon-build/images/freeswitch:${env.FS_VERSION} \
                                                                             europe-west1-docker.pkg.dev/libon-build/images/freeswitch:${releaseVersion} --project libon-build
                                        """
                                    }

                                } catch (e) {
                                    slackSend channel: '#team-voice',
                                              color: 'danger',
                                              message: "Release of FreeSWITCH\nFailure after ${currentBuild.durationString} (<${env.BUILD_URL}|Open>)"
                                    throw e
                                }
                            }
                        }
                    }
                }
                script {
                    currentBuild.description = "Release ${releaseVersion}"
                    slackSend channel: '#team-voice',
                              color: 'good',
                              message: "Release of FreeSWITCH ${releaseVersion}\nSuccess after ${currentBuild.durationString} (<${env.BUILD_URL}|Open>)"
                }
            }

        }
    }
}
