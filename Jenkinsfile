pipeline {
  options {
    ansiColor('xterm')
  }

  environment {
    ANSIBLE_VAULT = credentials('passwd_vars')
  }

  agent {
    kubernetes {
      cloud 'openshift'
      defaultContainer 'ansible'
      namespace 'cit-hiperautomacao'
      //idleMinutes 1
      yamlFile 'pod_ansible.yml'
    }
  }

  stages {
    stage('Deploy Virtual Machine(s)...') {
      steps {
        script {
          stage('Build Ansible Inventory') {
            sh '''#!/bin/bash
              cd /home/ansible

              echo 'localhost' >./hosts

              cat ./hosts
            '''
          }
          stage('Create the encrypted Ansible Vault at the Ansible POD') {
            sh '''#!/bin/bash
              cat "${ANSIBLE_VAULT}" >/home/ansible/.passwd_vars.yml
            '''
          }
          stage('Create Ansible Config at the Ansible POD') {
            sh '''#!/bin/bash
              mv ./ansible.cfg /home/ansible/.ansible.cfg
              cat /home/ansible/.ansible.cfg
            '''
          }
          stage('Clone Ansible Role(s) from the git server') {
            withCredentials([gitUsernamePassword(credentialsId:
              'gitcredentials', gitToolName: 'git')]) {
              sh '''#!/bin/bash
                mkdir /home/ansible/roles 2>/dev/null
                cd /home/ansible/roles
                echo $(pwd)
              '''
            }
          }
          stage('Execute Playbook') {
            sh '''#!/bin/bash
              git clone https://github.com/LeoFrancaBessa/test_procedure_ansible.git
              mv ./playbook.yml /home/ansible/main_playbook.yml
              cat /home/ansible/main_playbook.yml
            '''
            ansiblePlaybook (
              playbook: '/home/ansible/main_playbook.yml',
              inventory: '/home/ansible/hosts',
              vaultCredentialsId: 'ansible_vault_pass',
              colorized: 'true'
            )
          }
        }
      }
    }
  }

  post {
    always {
      cleanWs (
        cleanWhenAborted: true,
        cleanWhenFailure: true,
        cleanWhenNotBuilt: false,
        cleanWhenSuccess: true,
        cleanWhenUnstable: true,
        deleteDirs: true,
        notFailBuild: true,
        disableDeferredWipeout: true
      )
    }
  }
}
