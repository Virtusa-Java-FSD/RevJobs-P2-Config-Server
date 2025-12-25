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
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    powershell '''
                        # 1. Fix Key Permissions (Critical for Windows OpenSSH)
                        $keyPath = $env:SSH_KEY
                        $acl = Get-Acl $keyPath
                        $acl.SetAccessRuleProtection($true, $false) # Disable inheritance, remove existing rules
                        $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
                        $rule = New-Object System.Security.AccessControl.FileSystemAccessRule($currentUser, "FullControl", "Allow")
                        $acl.SetAccessRule($rule)
                        Set-Acl $keyPath $acl

                        # 2. Deployment Steps
                        $remote = "$env:REMOTE_USER@$env:REMOTE_HOST"
                        
                        # Stop existing service (ignoring errors)
                        ssh -i $keyPath -o StrictHostKeyChecking=no $remote "pkill -f config-server; exit 0"

                        # Create directory
                        ssh -i $keyPath -o StrictHostKeyChecking=no $remote "mkdir -p $env:REMOTE_DIR"

                        # Upload JAR
                        # Note: Resolving wildcard in PowerShell before passing to scp
                        $jarFile = Get-Item "target/*.jar"
                        scp -i $keyPath -o StrictHostKeyChecking=no $jarFile $remote":"$env:REMOTE_DIR/config-server.jar

                        # Start Service
                        ssh -i $keyPath -o StrictHostKeyChecking=no $remote "nohup java -jar $env:REMOTE_DIR/config-server.jar > $env:REMOTE_DIR/log.txt 2>&1 &"
                    '''
                }
        }
    }
}
