pipeline {
    agent any

    triggers {
        githubPush()
        pollSCM('H/5 * * * *')
    }

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
                    powershell '''
                        Write-Host "Starting deployment..."
                        
                        # 1. Fix Key Permissions (Critical for Windows OpenSSH)
                        Write-Host "Setting key permissions..."
                        $keyPath = $env:SSH_KEY
                        $acl = Get-Acl $keyPath
                        $acl.SetAccessRuleProtection($true, $false) # Disable inheritance, remove existing rules
                        $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
                        $rule = New-Object System.Security.AccessControl.FileSystemAccessRule($currentUser, "FullControl", "Allow")
                        $acl.SetAccessRule($rule)
                        Set-Acl $keyPath $acl
                        Write-Host "Key permissions set successfully"

                        # 2. Deployment Steps
                        $remote = "$env:REMOTE_USER@$env:REMOTE_HOST"
                        Write-Host "Deploying to $remote"
                        
                        # Stop existing service (ignoring errors)
                        Write-Host "Stopping existing service..."
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "pkill -f config-server; exit 0"

                        # Create directory
                        Write-Host "Creating directory..."
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "mkdir -p $env:REMOTE_DIR"

                        # Upload JAR
                        Write-Host "Uploading JAR file..."
                        $jarFile = Get-Item "target/*.jar"
                        scp -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $jarFile $remote":"$env:REMOTE_DIR/config-server.jar

                        # Start Service
                        Write-Host "Starting service..."
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "nohup java -jar $env:REMOTE_DIR/config-server.jar > $env:REMOTE_DIR/log.txt 2>&1 &"
                        
                        Write-Host "Deployment complete!"
                    '''
                }
            }
        }
    }
}
