pipeline {
    agent any

    triggers {
        githubPush()
        pollSCM('H/5 * * * *')
    }

    environment {
        REMOTE_HOST = "${env.EC2_INFRA_IP}"
        REMOTE_USER = "ec2-user"
        REMOTE_DIR = "/home/ec2-user/revjobs/revjob_p1_microservices/config-server/target"
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
                        
                        # 1. Fix Key Permissions
                        $keyPath = $env:SSH_KEY
                        $acl = Get-Acl $keyPath
                        $acl.SetAccessRuleProtection($true, $false)
                        $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
                        $rule = New-Object System.Security.AccessControl.FileSystemAccessRule($currentUser, "FullControl", "Allow")
                        $acl.SetAccessRule($rule)
                        Set-Acl $keyPath $acl

                        # 2. Deployment Steps
                        $remote = "$env:REMOTE_USER@$env:REMOTE_HOST"
                        
                        # Stop service
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "sudo systemctl stop config-server"

                        # Create directory
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "mkdir -p $env:REMOTE_DIR"

                        # Upload JAR
                        $jarFile = Get-Item "target/*.jar"
                        scp -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $jarFile $remote":"$env:REMOTE_DIR/config-server-1.0.0.jar

                        # Start Service
                        ssh -i $keyPath -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no $remote "sudo systemctl start config-server"
                        
                        Write-Host "Deployment complete!"
                    '''
                }
            }
        }
    }
}
