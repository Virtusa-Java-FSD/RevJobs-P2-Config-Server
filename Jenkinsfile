pipeline {
    agent any

    environment {
        REMOTE_HOST = "${env.EC2_INFRA_IP}"
        REMOTE_USER = "ec2-user"
        REMOTE_DIR = "/home/ec2-user/microservices/config-server" // Ensure this dir exists on EC2
        SSH_CREDENTIALS_ID = "ec2-ssh-key"
    }

    tools {
        maven 'Maven 3'
        jdk 'JDK 17'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                     // 1. Stop existing service
                     bat "ssh -i \"${SSH_KEY}\" -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'pkill -f config-server || true'"

                     // 2. Create directory if not exists
                     bat "ssh -i \"${SSH_KEY}\" -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'mkdir -p ${REMOTE_DIR}'"

                     // 3. Upload new JAR
                     bat "scp -i \"${SSH_KEY}\" -o StrictHostKeyChecking=no target/*.jar ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/config-server.jar"

                     // 4. Start Service in Background
                     bat "ssh -i \"${SSH_KEY}\" -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'nohup java -jar ${REMOTE_DIR}/config-server.jar > ${REMOTE_DIR}/log.txt 2>&1 &'"
                }
            }
        }
    }
}
