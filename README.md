# ansible-project
ansible-challenge

# Ansible CI Challenge – Terraform + Ansible + Jenkins 🚀

## 📌 Project Overview

This project demonstrates a **complete CI/CD automation workflow** using:

- **Terraform** – Infrastructure provisioning
- **Ansible** – Configuration management
- **Jenkins** – CI pipeline orchestration
- **AWS EC2** – Cloud infrastructure

The pipeline automatically:
1. Provisions EC2 instances using Terraform
2. Dynamically generates an Ansible inventory
3. Configures servers using Ansible roles
4. Sets up a reverse proxy using NGINX
5. Deploys Netdata monitoring on the backend
6. Executes everything end-to-end using Jenkins with zero manual steps

---

## 🎯 Challenge Requirements

### Infrastructure
- Deploy **2 EC2 instances**
  - **Amazon Linux 2**
    - Hostname: `c8.local`
    - Group: `frontend`
  - **Ubuntu 21.04**
    - Hostname: `u21.local`
    - Group: `backend`

### Configuration
- Disable **SELinux**
- Disable **Firewall**
- Install & configure **NGINX** on frontend
- Reverse proxy:
  - Port `80` → Backend `19999`
- Install **Netdata** on backend (port `19999`)
- No hardcoded IPs (dynamic inventory)

---

## 🛠 Prerequisites (Jenkins / Control Node)

Required tools:
```bash
terraform --version
ansible --version
git --version
aws --version
java -version
````

AWS requirements:

* IAM user with EC2 permissions
* Access key & secret key
* SSH key pair (example: `devops.pem`)

---

## 📁 Project Structure

```
devops-ci-challenge/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── inventory.tpl
│
├── ansible/
│   ├── site.yml
│   ├── inventory
│   └── roles/
│       ├── common/
│       ├── frontend/
│       └── backend/
│
└── Jenkinsfile
```

---

## ⚙️ STEP 1 – Terraform Configuration

### Terraform Responsibilities

* Create EC2 instances
* Assign hostnames
* Generate dynamic Ansible inventory

---

### Terraform Variables (`terraform/variables.tf`)

```hcl
variable "region" {
  default = "us-east-1"
}

variable "key_name" {
  default = "devops"
}
```

---

### Terraform Main (`terraform/main.tf`)

```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "frontend" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  key_name      = var.key_name

  tags = {
    Name = "c8.local"
  }
}

resource "aws_instance" "backend" {
  ami           = "ami-08c40ec9ead489470"
  instance_type = "t2.micro"
  key_name      = var.key_name

  tags = {
    Name = "u21.local"
  }
}
```

---

### Dynamic Inventory Template (`terraform/inventory.tpl`)

```ini
[frontend]
c8.local ansible_host=${frontend_ip} ansible_user=ec2-user

[backend]
u21.local ansible_host=${backend_ip} ansible_user=ubuntu
```

---

### Terraform Output & Inventory Generation (`terraform/outputs.tf`)

```hcl
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/inventory.tpl", {
    frontend_ip = aws_instance.frontend.public_ip
    backend_ip  = aws_instance.backend.public_ip
  })

  filename = "${path.module}/../ansible/inventory"
}
```

---

## ⚙️ STEP 2 – Ansible Configuration (Roles)

### Create Roles

```bash
ansible-galaxy init roles/common
ansible-galaxy init roles/frontend
ansible-galaxy init roles/backend
```

---

### Common Role (`roles/common/tasks/main.yml`)

```yaml
- name: Disable SELinux
  selinux:
    state: disabled
  when: ansible_os_family == "RedHat"

- name: Stop firewalld
  service:
    name: firewalld
    state: stopped
    enabled: false
  when: ansible_os_family == "RedHat"

- name: Stop UFW
  service:
    name: ufw
    state: stopped
    enabled: false
  when: ansible_os_family == "Debian"
```

---

### Frontend Role – NGINX (`roles/frontend/tasks/main.yml`)

```yaml
- name: Install Nginx
  package:
    name: nginx
    state: present

- name: Configure Nginx reverse proxy
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf

- name: Start Nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

#### NGINX Template (`roles/frontend/templates/nginx.conf.j2`)

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      proxy_pass http://{{ groups['backend'][0] }}:19999;
    }
  }
}
```

✔ Uses Ansible inventory dynamically
✔ No hardcoded IPs

---

### Backend Role – Netdata (`roles/backend/tasks/main.yml`)

```yaml
- name: Install Netdata
  shell: curl -s https://my-netdata.io/kickstart.sh | bash
  args:
    creates: /usr/sbin/netdata

- name: Ensure Netdata running
  service:
    name: netdata
    state: started
    enabled: yes
```

---

## 📄 Main Playbook (`ansible/site.yml`)

```yaml
---
- hosts: all
  become: yes
  roles:
    - common

- hosts: frontend
  become: yes
  roles:
    - frontend

- hosts: backend
  become: yes
  roles:
    - backend
```

---

## 🤖 STEP 3 – Jenkins CI Pipeline

### Jenkins Responsibilities

* Run Terraform
* Generate inventory
* Execute Ansible playbooks automatically

---

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "us-east-1"
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/shaikhshahbazz/ansible-project.git', branch: 'main'
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
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    dir('terraform') {
                        sh 'terraform apply -auto-approve'
                    }
                }
            }
        }

        stage('Ansible Configure') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-private-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    dir('ansible') {
                        sh '''
                        chmod 600 $SSH_KEY
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i inventory site.yml \
                        --private-key=$SSH_KEY
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
```

---

## 🔐 Jenkins Credentials

Store in Jenkins:

* **AWS_ACCESS_KEY_ID**
* **AWS_SECRET_ACCESS_KEY**
* **SSH Private Key**

---

## ✅ Final Validation

Access frontend:

```
http://<frontend-public-ip>
```

✔ NGINX responds
✔ Netdata UI loads
✔ Reverse proxy works

---

## 🧠 Key Learnings

* Terraform + Ansible integration
* Dynamic inventory generation
* Role-based Ansible architecture
* Jenkins CI automation
* Real-world DevOps workflow

---

## 👨‍💻 Author

**Shaikh Shahbaz**
DevOps | Terraform | Ansible | Jenkins | AWS

```


