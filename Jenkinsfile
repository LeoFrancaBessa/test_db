pipeline {
  options {
    ansiColor('xterm')
  }

  environment {
    ANSIBLE_VAULT = credentials('passwd_vars')
     // Variáveis do repositório e credenciais
      GIT_REPO = 'https://github.com/LeoFrancaBessa/test_cam.git'
      BRANCH = 'main'
      GIT_AUTHOR_USERNAME = 'teste'
      GIT_AUTHOR_EMAIL = 'teste@teste.com'
      commitMessage = "Adicionado arquivo da procedure SQL"

     // Variaveis banco de dados
      DB_HOST = "10.1.1.80"
      DB_NAME = "dev"
      DB_PORT = '1521'

      // Definir o conteúdo da procedure diretamente como texto
      PACKAGE_NAME = "exemplo_package"
      PACKAGE_HEAD = """CREATE OR REPLACE PACKAGE HAUT.exemplo_package IS
                      -- Procedimento que imprime uma mensagem
                      PROCEDURE diga_ola(nome IN VARCHAR2);

                      -- Função que retorna uma saudação personalizada
                      FUNCTION saudacao(nome IN VARCHAR2) RETURN VARCHAR2;
                      END exemplo_package;"""
      
      PACKAGE_BODY = """CREATE OR REPLACE PACKAGE BODY HAUT.exemplo_package IS
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
                      END exemplo_package;""".replace("'", "''")
  }

  agent {
    kubernetes {
      cloud 'openshift'
      defaultContainer 'ansible'
      namespace 'cit-hiperautomacao'
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
            withCredentials([usernamePassword(credentialsId: 'dev-bd-credentials', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
              ansiblePlaybook (
                playbook: '/home/ansible/main_playbook.yml',
                inventory: '/home/ansible/hosts',
                vaultCredentialsId: 'ansible_vault_pass',
                colorized: 'true',
                extraVars: [
                  db_host : "${DB_HOST}",
                  db_name : "${DB_NAME}",
                  db_port: "${DB_PORT}",
                  db_user : "${DB_USER}",
                  db_pass : "${DB_PASS}",
                  package_head : '${PACKAGE_HEAD}',
                  package_body : '${PACKAGE_BODY}'
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
          stage('Create SQL File with Echo') {
            script {
              sh """
                echo "${PACKAGE_HEAD}" > /tmp/test_cam_repo/test_cam/${PACKAGE_NAME}.sql
                echo "/" >> /tmp/test_cam_repo/test_cam/${PACKAGE_NAME}.sql
                echo "${PACKAGE_BODY}" >> /tmp/test_cam_repo/test_cam/${PACKAGE_NAME}.sql
                echo "/" >> /tmp/test_cam_repo/test_cam/${PACKAGE_NAME}.sql
              """
            }
          }
          stage('Commit and Push Changes to Git') {
            withCredentials([usernamePassword(credentialsId: 'github-test-procedure', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh """
                  git config --global user.email "${GIT_AUTHOR_EMAIL}"
                  git config --global user.name "${GIT_AUTHOR_USERNAME}"
                  cd /tmp/test_cam_repo/test_cam
                  git add .

                  # Verificar se há mudanças staged
                  if git diff --cached --exit-code; then
                      echo "Sem modificações para commit."
                  else
                      git commit -m "${commitMessage}"
                      git pull --rebase https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LeoFrancaBessa/test_cam.git ${BRANCH} || true
                      git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/LeoFrancaBessa/test_cam.git ${BRANCH}
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
        // Move o log para o workspace
        sh "mv /home/ansible/log.txt ${env.WORKSPACE}/log.txt"
    }

    success {
        script {
            sendNotification('SUCESSO')
        }
    }

    failure {
        script {
            sendNotification('FALHOU')
        }
    }

    aborted {
        script {
            sendNotification('ABORTADO')
        }
    }
  }
}


def sendNotification(String status) {
// Constrói a mensagem com o status diretamente
def message = "{\"content\": \"Deploy ${status} na base ${DB_HOST}, schema ${DB_NAME} \\nLog completo: http://jenkins.sefaz.ma.gov.br/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/console \\nCommit: ${commitMessage} \\nAutor: ${GIT_AUTHOR_USERNAME} \\nLog da Execução SQL Plus: \"}"

// Envia a notificação via HTTP Request
httpRequest httpMode: 'POST', 
    url: 'https://discordapp.com/api/webhooks/1296172490657234966/eS1biobe9Ll34r-lf4VSHcw4kALMslJa7CuN0V485vXy2sZCauM00szX4Lzjq-H6xuhs',
    formData: [
      [contentType: 'application/json', name: 'payload_json', body: message],
      [contentType: 'text/plain', name: 'file1', fileName: 'log.txt', uploadFile: "${env.WORKSPACE}/log.txt"]
  ]
}