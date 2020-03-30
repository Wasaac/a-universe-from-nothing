pipeline {
    options { disableConcurrentBuilds() }
    agent { label 'docker' }
    parameters {
        credentials credentialType: 'com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey', defaultValue: '', description: 'Kayobe SSH Key', name: 'KAYOBE_SSH_CREDS', required: true
        password defaultValue: 'SECRET', description: 'Kayobe Ansible Vault Password', name: 'KAYOBE_VAULT_PASSWORD'
        string defaultValue: 'http://192.168.33.5:4000/', description: 'Docker Registry to push images to', name: 'DOCKER_REGISTRY', trim: true
        string defaultValue: '6.0.0', description: 'Kayobe version tag to use', name: 'KAYOBE_VERSION', trim: true
        string defaultValue: 'https://git.openstack.org/openstack/kayobe.git', description: 'Kayobe repo to use', name: 'KAYOBE_REPO', trim: true
        credentials credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.FileCredentialsImpl', description: 'Kayobe SSH Config file', name: 'KAYOBE_SSH_CONFIG', required: false
    }
    environment {
        REGISTRY = "${params.DOCKER_REGISTRY}"
        KAYOBE_VERSION = "${params.KAYOBE_VERSION}"
        KAYOBE_REPO = "${params.KAYOBE_REPO}"
        KAYOBE_IMAGE = "${currentBuild.projectName}:${env.GIT_COMMIT}"
    }
    stages {
        stage('Build and Push') {
            steps {
                script {
                    def kayobeImage = docker.build("$KAYOBE_IMAGE",
		    "--build-arg kayobe_version=$KAYOBE_VERSION --build-arg kayobe_repo=$KAYOBE_REPO .")
                    docker.withRegistry("$REGISTRY") {
                        kayobeImage.push()
                        kayobeImage.push('latest')
                    }
                }
            }
        }
        stage('Deploy') {
            stages {
                stage('Prepare Secrets') {
                    environment {
                        KAYOBE_VAULT_PASSWORD = "${params.KAYOBE_VAULT_PASSWORD}"
                        KAYOBE_SSH_CREDS_FILE = credentials("${params.KAYOBE_SSH_CREDS}")
                    }
                    steps {
                        sh 'mkdir -p secrets/.ssh'
                        sh "cp $KAYOBE_SSH_CREDS_FILE secrets/.ssh/id_rsa"
                        sh(returnStdout: false, script: 'ssh-keygen -y -f secrets/.ssh/id_rsa > secrets/.ssh/id_rsa.pub')
                        sh(returnStdout: false, script: 'echo $KAYOBE_VAULT_PASSWORD > secrets/vault.pass')
                    }
                }
                stage('Optionally prepare SSH Config') {
                    when {
                        expression { params.KAYOBE_SSH_CONFIG != null && params.KAYOBE_SSH_CONFIG != '' }
                    }
                    environment {
                        KAYOBE_SSH_CONFIG_FILE = credentials("${params.KAYOBE_SSH_CONFIG}")
                    }
                    steps {
                        sh "cp $KAYOBE_SSH_CONFIG_FILE secrets/.ssh/config"
                    }
                }
                stage('Run Kayobe') {
                    agent {
                        docker {
                            image "$KAYOBE_IMAGE"
                            registryUrl "$REGISTRY"
                            reuseNode true
                        }
                    }
                    environment {
                        KAYOBE_VAULT_PASSWORD = "${params.KAYOBE_VAULT_PASSWORD}"
                    }
                    steps {
                        sh 'cp -R secrets/. /secrets'
                        sh '/bin/entrypoint.sh echo READY'
                        sh 'kayobe control host bootstrap'
                        sh 'kayobe overcloud inventory discover'
                        sh 'kayobe overcloud service reconfigure'
                    }
                }
            }
        }
    }
    post {
        cleanup {
            dir('secrets') {
                deleteDir()
            }
        }
   }
}
