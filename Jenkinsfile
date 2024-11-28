pipeline {
    agent any
    
    environment {
        // Semgrep credentials
        SEMGREP_APP_TOKEN = credentials('SEMGREP_APP_TOKEN')
        
        // Deployment details
        PRODUCTION_IP_ADDRESS = '52.43.163.218'
        APP_PORT = '3000'
        
        // Nuclei scan configurations
        NUCLEI_TEMPLATES_PATH = '/home/ec2-user/nuclei-templates/'
    }
    
    tools {
        nodejs "nodejs"
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                script {
                    // Install global tools
                    sh 'npm install -g yarn pm2'
                    sh 'pip3 install semgrep'
                    
                    // Install project dependencies
                    sh 'yarn install'
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Build steps (if any)
                    sh 'yarn build || true'
                }
            }
        }
        
        stage('SAST - Semgrep Scan') {
            steps {
                script {
                    try {
                        // Run Semgrep scan
                        sh '''
                            /var/lib/jenkins/.local/bin/semgrep ci \
                            --config=auto \
                            --json \
                            --no-suppress-errors \
                            -o semgrep-results.json
                        '''
                    } catch (Exception e) {
                        echo "Semgrep scan completed with issues: ${e.message}"
                    }
                }
            }
            post {
                always {
                    // Archive Semgrep results
                    archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
                }
            }
        }
        
        stage('Deploy to Production') {
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
        
        stage('DAST - Nuclei Scan') {
            steps {
                script {
                    // Install Nuclei if not already installed
                    sh '''
                        which nuclei || {
                            wget https://github.com/projectdiscovery/nuclei/releases/download/v3.1.3/nuclei_3.1.3_linux_amd64.zip
                            unzip nuclei_3.1.3_linux_amd64.zip
                            sudo mv nuclei /usr/local/bin/
                        }
                        
                        # Perform Nuclei scan
                        nuclei -u http://${PRODUCTION_IP_ADDRESS}:${APP_PORT} \
                               -t ${NUCLEI_TEMPLATES_PATH} \
                               -o nuclei-results.txt
                    '''
                }
            }
            post {
                always {
                    // Archive Nuclei scan results
                    archiveArtifacts artifacts: 'nuclei-results.txt', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            // Cleanup and notification steps
            echo "Pipeline execution completed"
            
            // Optional: Send notifications based on build status
            script {
                if (currentBuild.result == 'FAILURE') {
                    echo "Build failed. Sending notifications..."
                    // Add email or Slack notification logic here
                }
            }
        }
        
    }
}
