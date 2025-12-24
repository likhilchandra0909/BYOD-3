pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = "true"
        TF_CLI_ARGS = "-no-color"
        SSH_CRED_ID = "ssh-key-dev"
    }

    stages {

        stage('Terraform Apply') {
            steps {
                withCredentials([usernamePassword(
                credentialsId: 'aws-creds',
                usernameVariable: 'AWS_ACCESS_KEY_ID',
                passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh '''
                  terraform init
                  terraform apply -auto-approve -var-file=${BRANCH_NAME}.tfvars
            '''
            }
        }
    }


        stage('Capture Terraform Outputs') {
            steps {
                script {
                    env.INSTANCE_IP = sh(
                        script: "terraform output -raw instance_public_ip",
                        returnStdout: true
                    ).trim()

                    env.INSTANCE_ID = sh(
                        script: "terraform output -raw instance_id",
                        returnStdout: true
                    ).trim()

                    echo "IP: ${env.INSTANCE_IP}"
                    echo "ID: ${env.INSTANCE_ID}"
                }
            }
        }

        stage('Create Dynamic Inventory') {
            steps {
                sh '''
                  echo "[splunk]" > dynamic_inventory.ini
                  echo "${INSTANCE_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa" >> dynamic_inventory.ini
                '''
            }
        }

        stage('Wait for EC2 Health') {
            steps {
                sh '''
                  aws ec2 wait instance-status-ok --instance-ids ${INSTANCE_ID}
                '''
            }
        }

        stage('Install Splunk') {
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/splunk.yml',
                    inventory: 'dynamic_inventory.ini'
                )
            }
        }

        stage('Test Splunk') {
            steps {
                ansiblePlaybook(
                    playbook: 'playbooks/test-splunk.yml',
                    inventory: 'dynamic_inventory.ini'
                )
            }
        }

        stage('Validate Destroy') {
            steps {
                input message: "Destroy infrastructure?"
            }
        }

        stage('Terraform Destroy') {
            steps {
                sh '''
                  terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f dynamic_inventory.ini'
        }

        failure {
            sh 'terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars'
        }

        aborted {
            sh 'terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars'
        }
    }
}
