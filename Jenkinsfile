pipeline {
    agent any
    environment {
        PRODUCTION_IP_ADDRESS = '52.43.163.218'
    }
    
    tools {
        nodejs "nodejs"
    }
    
    stages {
        stage('Local Preparation') {
            steps {
                script {
                    sh 'npm install -g yarn pm2'
                    sh 'yarn install'
                }
            }
        }
        
        stage('Deploy to EC2') {
            environment {
                DEPLOY_SSH_KEY = credentials('aws_ssh_key')
            }
            steps {
                script {
                    sh '''
                        chmod 600 $DEPLOY_SSH_KEY
                        ssh -o StrictHostKeyChecking=no -i $DEPLOY_SSH_KEY ec2-user@$PRODUCTION_IP_ADDRESS '
                            # Install Node.js and dependencies
                            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
                            source ~/.bashrc
                            
                            # Install latest LTS Node.js
                            nvm install --lts
                            
                            # Install Yarn and PM2 globally
                            npm install -g yarn pm2
                            
                            # Ensure Git is installed
                            sudo yum update -y
                            sudo yum install -y git
                            
                            # Clone or Update the repository
                            if [ ! -d "My_codes" ]; then
                                git clone https://github.com/sagarmondi/My_codes.git
                                cd My_codes
                            else
                                cd My_codes
                                git pull
                            fi
                            
                            # Install project dependencies
                            yarn install
                            
                            # Start or Restart the application
                            if pm2 describe todos-app > /dev/null 2>&1; then
                                pm2 restart todos-app
                            else
                                pm2 start --name todos-app npm -- start
                            fi
                        '
                    '''
                }
            }
        }
    }
    
    post {
        failure {
            echo "Deployment failed. Check the logs for detailed information."
        }
        success {
            echo "Deployment completed successfully!"
        }
    }
}
