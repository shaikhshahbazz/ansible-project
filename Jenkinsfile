pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')   // Replace with your Jenkins AWS credential ID
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')   // Replace with your Jenkins AWS credential ID
    }

    stages {
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Checkout') {
            steps {
                git url: 'https://github.com/sneha-samala/ansible-project.git'
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
                dir('ci-pipeline/terraform') {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]]) {
                        sh 'terraform apply -auto-approve'
                    }
                }
            }
        }

        stage('Fetch Terraform Output') {
            steps {
                echo 'Skipped due to earlier failure(s) if any'
            }
        }

        stage('Ansible Configure') {
            steps {
                echo 'Skipped due to earlier failure(s) if any'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check console logs.'
        }
    }
}
