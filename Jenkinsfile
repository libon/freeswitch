pipeline {
    agent none
    options {
        skipStagesAfterUnstable()
        ansiColor('xterm')
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
                        cd /

                        git clone https://github.com/libon/mod_prometheus.git

                        cd /mod_prometheus && git checkout ${params.MOD_PROMETHEUS_VERSION}

                        cargo build

                        cp /mod_prometheus/target/debug/libmod_prometheus.so /tmp/mod_prometheus.so
                    """
                }
                stash(name: "mod_prometheus", includes: "/tmp/mod_prometheus.so")
                milestone ordinal: 10, label: 'Building mod_prometheus'
            }
        }
        stage('Build freeswitch') {
            agent {
                kubernetes {
                    label "freeswitch-build"
                    defaultContainer 'build'
                    yamlFile 'libon/jenkins/freeswitch-build-stage.yaml'
                    yamlMergeStrategy([$class: 'org.csanchez.jenkins.plugins.kubernetes.pod.yaml.Merge'])
                }
            }
            steps {
                script {
                    sh """
                        BRANCH=`git branch --show-current`
                        if [ ${BRANCH} == "master" ]; then
                            COMMIT=`git rev-parse HEAD`
                            FS_VERSION="master-${COMMIT}"
                        else
                            FS_VERSION=`git describe --tags`
                        fi
                        
                        echo -n \$FS_VERSION > version
                        
                        git config --global --add safe.directory `pwd`

                        ./bootstrap.sh -j

                        ./configure --prefix=/opt/freeswitch

                        make && make install && make sounds-install && make moh-install

                        tar -zcvf /tmp/freeswitch.tar.gz /opt/freeswitch
                    """
                    env.FS_VERSION = sh(returnStdout: true, script: "cat version").trim()
                }
                stash(name: "freeswitch_bin", includes: "/tmp/freeswitch.tar.gz")
                milestone ordinal: 20, label: 'Building FreeSWITCH'
            }
        }
        stage('Build/push docker image') {
            agent {
                kubernetes {
                    label "image-build"
                    yamlFile 'libon/jenkins/image-build-stage.yaml'
                    yamlMergeStrategy([$class: 'org.csanchez.jenkins.plugins.kubernetes.pod.yaml.Merge'])
                }
            }
            steps {
                unstash(name: "mod_prometheus")
                unstash(name: "freeswitch_bin")
                container(name:"kaniko", shell: '/busybox/sh') {
                    withEnv(['PATH+EXTRA=/busybox:/kaniko']) {
                        sh """#!/busybox/sh
                        /kaniko/executor --context `pwd` \
                            --dockerfile=libon/docker/Dockerfile \
                            --destination=europe-west1-docker.pkg.dev/libon-build/images/freeswitch:${env.FS_VERSION}
                        """
                    }
                }
                milestone ordinal: 30, label: 'Building image'
            }
        }
    }
}
