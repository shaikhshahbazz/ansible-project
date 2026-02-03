pipeline {
  agent any

  environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
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
      environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION    = 'us-east-1'
      }
      steps {
        dir('ci-pipeline/terraform') {
          sh 'terraform apply -auto-approve'
        }
      }
    }

    stage('Ansible Configure') {
      steps {
        dir('ci-pipeline/ansible') {
          sh 'ansible-playbook -i inventory site.yml'
        }
      }
    }
  }
}
