pipeline {
  agent any

  environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
    SSH_USER = 'ec2-user'
  }

  stages {

    stage('Checkout') {
      steps {
        git 'https://github.com/shaikhshahbazz/ansible-project.git'
      }
    }

    stage('Terraform Init') {
      steps {
        dir('ci-pipeline/terraform') {
          sh 'terraform init'
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          dir('ci-pipeline/terraform') {
            sh 'terraform apply -auto-approve'
          }
        }
      }
    }

    stage('Prepare Ansible Variables') {
      steps {
        dir('ci-pipeline/ansible') {
          sh '''
          BACKEND_IP=$(awk '/\\[backend\\]/{getline; print $1}' inventory)
          echo "Backend IP detected: $BACKEND_IP"
          echo "BACKEND_IP=$BACKEND_IP" > backend.env
          '''
        }
      }
    }

    stage('Ansible Configure') {
      steps {
        withCredentials([
          sshUserPrivateKey(
            credentialsId: 'ansible-ssh-key',
            keyFileVariable: 'SSH_KEY'
          )
        ]) {
          dir('ci-pipeline/ansible') {
            sh '''
            source backend.env
            chmod 600 $SSH_KEY
            ansible-playbook -i inventory site.yml \
              --user=$SSH_USER \
              --private-key=$SSH_KEY \
              --extra-vars "backend_ip=$BACKEND_IP"
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo '✅ Pipeline completed successfully!'
    }
    failure {
      echo '❌ Pipeline failed. Check logs above.'
    }
  }
}
