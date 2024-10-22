pipeline {
  options {
    ansiColor('xterm')
  }

  environment {
    ANSIBLE_VAULT = credentials('passwd_vars')
     // Variáveis do repositório e credenciais
      GIT_REPO = 'https://github.com/LeoFrancaBessa/test_cam.git'
      BRANCH = 'teste'
      GIT_AUTHOR_USERNAME = 'teste'
      GIT_AUTHOR_EMAIL = 'teste@teste.com'
      commitMessage = "Adicionado arquivo da procedure SQL"

      // Definir o conteúdo da procedure diretamente como texto
      PACKAGE_HEAD = """CREATE OR REPLACE PACKAGE exemplo_package IS
                          -- Procedimento que imprime uma mensagem
                          PROCEDURE diga_ola(nome IN VARCHAR2);

                          -- Função que retorna uma saudação personalizada
                          FUNCTION saudacao(nome IN VARCHAR2) RETURN VARCHAR2;
                        END exemplo_package;"""
      
      PACKAGE_BODY = """CREATE OR REPLACE PACKAGE BODY exemplo_package IS
                        -- Implementação do procedimento que imprime uma mensagem
                        PROCEDURE diga_ola(nome IN VARCHAR2) IS
                        BEGIN
                            DBMS_OUTPUT.PUT_LINE('Olá, ' || nome || '!');
                        END diga_ola;

                        -- Implementação da função que retorna uma saudação personalizada
                        FUNCTION saudacao(nome IN VARCHAR2) RETURN VARCHAR2 IS
                        BEGIN
                            RETURN 'Saudações, ' || nome || '!';
                        END saudacao;
                      END exemplo_package;"""
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
            sh '''#!/bin/bash
              mkdir /home/ansible/roles 2>/dev/null
              cd /home/ansible/roles
              echo $(pwd)
              git clone https://github.com/LeoFrancaBessa/test_procedure_ansible.git test_procedure_ansible
            '''
          }
          stage('Execute Playbook') {
            sh '''#!/bin/bash
              mv ./playbook.yml /home/ansible/main_playbook.yml
              cat /home/ansible/main_playbook.yml
            '''
            withCredentials([usernamePassword(credentialsId: 'github-test-procedure', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
              ansiblePlaybook (
                playbook: '/home/ansible/main_playbook.yml',
                inventory: '/home/ansible/hosts',
                vaultCredentialsId: 'ansible_vault_pass',
                colorized: 'true',
                extraVars: [
                  db_host : "10.1.1.80",
                  db_name : "dev",
                  db_port: '1521',
                  db_user : "${DB_USER}",
                  db_pass : "${DB_PASS}"
                  package_head : "${PACKAGE_HEAD}",
                  package_body : "${PACKAGE_BODY}"
                ]
              )
            }
          }
          stage('Clone Packages Repository') {
            sh """
                mkdir -p /tmp/test_cam_repo  // Cria um diretório temporário para o repositório
                cd /tmp/test_cam_repo        // Muda para o diretório temporário
                git clone ${GIT_REPO}
                cd test_cam
                git switch ${BRANCH} || git switch -c ${BRANCH}
            """
          }
          stage('Commit and Push Changes to Git') {
            withCredentials([usernamePassword(credentialsId: 'github-test-procedure', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh """
                  git config --global user.email "${GIT_AUTHOR_EMAIL}"
                  git config --global user.name "${GIT_AUTHOR_USERNAME}"
                  cd /tmp/test_cam_repo/test_cam

                  # Verificar se há mudanças no repositório
                  if ! git diff --quiet HEAD; then
                      git add .
                      git commit -m "${commitMessage}"
                      git pull --rebase https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LeoFrancaBessa/test_db.git ${BRANCH} || true
                      git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LeoFrancaBessa/test_db.git ${BRANCH}
                  else
                      echo "Sem mudanças para commitar."
                  fi
              """
            }
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
