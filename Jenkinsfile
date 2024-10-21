properties([
  parameters([
    [
      $class: 'ChoiceParameter',
      choiceType: 'PT_SINGLE_SELECT',
      name: 'DB_NAME',
      description: "Banco de dados.",
      script: [
        $class: 'GroovyScript',
        script: [
          classpath: [],
          sandbox: true,
          script: '''
            return [
              "":"*** ESCOLHA ***",
              "arma":"arma",
              "basebi.sefaz.ma.gov.br":"basebi.sefaz.ma.gov.br",
              "cent":"cent",
              "viva":"viva"
            ]
          '''.stripIndent()
        ],
        fallbackScript: [
          classpath: [],
          sandbox: false,
          script:
            'return [\'error\']'
        ]
      ]
    ],
    [
      $class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'Nome do usuário.',
      name: 'USERNAME_DB',
//      referencedParameters: 'ACTIVITY',
      script: [
        $class: 'GroovyScript',
        fallbackScript: [
          classpath: [],
          sandbox: true,
          script: '''
            return '*** PARÂMETRO NÃO UTILIZADO ***'
          '''
        ],
        script: [
          classpath: [],
          sandbox: true,
          script: '''
            inputBox='<input class="setting-input" name="value" type="text" \
              value="">'
          '''
        ]
      ]
    ],
    [
      $class: 'ChoiceParameter',
      choiceType: 'PT_SINGLE_SELECT',
      name: 'PROCEDURE',
      description: "Ação a ser executada.",
      script: [
        $class: 'GroovyScript',
        script: [
          classpath: [],
          sandbox: true,
          script: '''
            return [
              "":"*** ESCOLHA ***",
              "create_user":"Criar usuário",
              "lock_user":"Bloquear usuário",
              "unlock_session":"Finalizar sessão de usuário",
              "unlock_user":"Desbloquear usuário"
            ]
          '''.stripIndent()
        ],
        fallbackScript: [
          classpath: [],
          sandbox: false,
          script:
            'return [\'error\']'
        ]
      ]
    ],
    string(defaultValue: '', name: 'TICKET', description: 'Ticket',
      trim: true ),
    string(defaultValue: '', name: 'LOGGED_USER',
      description: 'Nome do Usuario', trim: true),
    string(defaultValue: '', name: 'LOGGED_MAIL', description: 'Email',
      trim: true )
  ])
])

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

                git clone http://git.sefaz.ma.gov.br/cit-haut/\
ansible-mgmt_db.git mgmt_db
                git clone http://git.sefaz.ma.gov.br/cit-haut/\
notificationMail.git
              '''
            }
          }
          stage('Execute Playbook') {
            sh '''#!/bin/bash
              mv ./playbook.yml /home/ansible/main_playbook.yml
              cat /home/ansible/main_playbook.yml
            '''
            ansiblePlaybook (
              playbook: '/home/ansible/main_playbook.yml',
              inventory: '/home/ansible/hosts',
              vaultCredentialsId: 'ansible_vault_pass',
              colorized: 'true',
              extraVars: [
                ticket_id: "${TICKET}",
                db_name: "${DB_NAME}",
                username_db: "${USERNAME_DB}",
                procedure: "${PROCEDURE}"
              ]/*,
              extras: '-vvv',
              tags: 'debug'*/
            )
          }
         stage('Execute Playbook: notificationMail') {
            sh '''#!/bin/bash
              mv ./notificationMail_playbook.yml /home/ansible/
              cat /home/ansible/notificationMail_playbook.yml
            '''
            ansiblePlaybook (
              playbook: '/home/ansible/notificationMail_playbook.yml',
              inventory: '/home/ansible/hosts',
              vaultCredentialsId: 'ansible_vault_pass',
              colorized: 'true',
              extraVars: [
                project_name: "mgmt_db",
                user_email: "${LOGGED_MAIL}",
                ticket_id: "${TICKET}",
                logged_user: "${LOGGED_USER}"
              ]/*,
              extras: '-vvvvv',
              tags: 'debug'*/
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
