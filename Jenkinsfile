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
        
        stage('SAST - Semgrep Scan and Rule Extraction') {
            steps {
                script {
                    // Run Semgrep scan with detailed output
                    def semgrepScanCommand = '''
                        /var/lib/jenkins/.local/bin/semgrep ci \
                        --json \
                        --no-suppress-errors \
                        -o semgrep-results.json
                    '''
                    
                    def semgrepResult = sh(
                        script: semgrepScanCommand,
                        returnStatus: true,
                        returnStdout: true
                    )
                    
                    // Extract rule information using shell commands
                    sh '''
                        # Extract unique rule IDs using grep and sort
                        grep -o '"rule_id":"[^"]*' semgrep-results.json | 
                        cut -d':' -f2 | 
                        sed 's/"//g' | 
                        sort | 
                        uniq > semgrep-used-rules.txt
                    '''
                    
                    // Read the extracted rules
                    def usedRules = readFile('semgrep-used-rules.txt').trim().split('\n')
                    
                    // Log the number of used rules
                    echo "Total Semgrep Rules Used: ${usedRules.size()}"
                    
                    // Print the rules
                    echo "Used Rules:"
                    usedRules.each { rule ->
                        echo "- ${rule}"
                    }
                    
                    if (semgrepResult != 0) {
                        error "Semgrep scan failed with exit code ${semgrepResult}"
                    }
                }
            }
            post {
                always {
                    // Archive Semgrep results and used rules
                    archiveArtifacts artifacts: 'semgrep-results.json,semgrep-used-rules.txt', allowEmptyArchive: true
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
        stage('DAST - Nuclei Scan and Rule Extraction') {
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
                        
                        # Extract used templates
                        grep -oP '(?<=\\[).*?(?=\\])' nuclei-results.txt | 
                        sort | 
                        uniq > nuclei-used-templates.txt
                    '''
                    
                    // Read the extracted templates
                    def usedTemplates = readFile('nuclei-used-templates.txt').trim().split('\n')
                    
                    // Log the number of used templates
                    echo "Total Nuclei Templates Used: ${usedTemplates.size()}"
                    
                    // Print the templates
                    echo "Used Templates:"
                    usedTemplates.each { template ->
                        echo "- ${template}"
                    }
                }
            }
            post {
                always {
                    // Archive Nuclei scan results and used templates
                    archiveArtifacts artifacts: 'nuclei-results.txt,nuclei-used-templates.txt', allowEmptyArchive: true
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
    }
    
    post {
        always {
            // Cleanup and notification steps
            echo "Pipeline execution completed"
            
            // Optional: Send notifications based on build status
            script {
                if (currentBuild.result == 'FAILURE') {
                    echo "Build failed. Sending notifications..."
                }
            }
        }
        cleanup {
            // Clean up workspace
            cleanWs()
        }
    }
}
