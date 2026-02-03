pipeline {
  agent any

  stages {

    stage('Checkout') {
      steps {
        git 'https://github.com/shaikhshahbazz/ansible-project.git'
      }
    }

    stage('Terraform Init') {
      steps {
        dir('terraform') {
          sh 'terraform init'
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        dir('terraform') {
          sh 'terraform apply -auto-approve'
        }
      }
    }

    stage('Ansible Configure') {
      steps {
        dir('ansible') {
          sh 'ansible-playbook -i inventory site.yml'
        }
      }
    }
  }
}

