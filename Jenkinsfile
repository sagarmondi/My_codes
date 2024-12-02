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
                    def semgrepResult = sh(
                        script: '''
                            /var/lib/jenkins/.local/bin/semgrep ci \
                            --json \
                            --no-suppress-errors \
                            -o semgrep-results.json
                        ''',
                        returnStatus: true,
                        returnStdout: true
                    )
                    
                    // Groovy-based rule extraction without external tools
                    def semgrepResults = readFile('semgrep-results.json')
                    
                    // Parse JSON and extract unique rule IDs
                    def jsonSlurper = new groovy.json.JsonSlurperClassic()
                    def parsedResults = jsonSlurper.parseText(semgrepResults)
                    
                    // Extract unique rule IDs
                    def usedRules = parsedResults.results.collect { it.extra.rule_id }.unique()
                    
                    // Write used rules to a file
                    def rulesJsonBuilder = new groovy.json.JsonBuilder()
                    rulesJsonBuilder(usedRules)
                    
                    writeFile(
                        file: 'semgrep-used-rules.json', 
                        text: rulesJsonBuilder.toPrettyString()
                    )
                    
                    // Log the number of used rules
                    echo "Total Semgrep Rules Used: ${usedRules.size()}"
                    
                    if (semgrepResult != 0) {
                        error "Semgrep scan failed with exit code ${semgrepResult}"
                    }
                }
            }
            post {
                always {
                    // Archive Semgrep results and used rules
                    archiveArtifacts artifacts: 'semgrep-results.json,semgrep-used-rules.json', allowEmptyArchive: true
                }
                failure {
                    echo "Semgrep scan stage failed"
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
                    '''
                    
                    // Read Nuclei results and extract used templates
                    def nucleiResults = readFile('nuclei-results.txt')
                    
                    // Use Groovy regex to extract template names
                    def templateMatcher = nucleiResults =~ /\[([^\]]+)\]/
                    def usedTemplates = []
                    
                    while (templateMatcher.find()) {
                        usedTemplates << templateMatcher.group(1)
                    }
                    
                    // Remove duplicates and write to file
                    def uniqueTemplates = usedTemplates.unique()
                    
                    writeFile(
                        file: 'nuclei-used-templates.json', 
                        text: new groovy.json.JsonBuilder(uniqueTemplates).toPrettyString()
                    )
                    
                    // Log the number of used templates
                    echo "Total Nuclei Templates Used: ${uniqueTemplates.size()}"
                }
            }
            post {
                always {
                    // Archive Nuclei scan results and used templates
                    archiveArtifacts artifacts: 'nuclei-results.txt,nuclei-used-templates.json', allowEmptyArchive: true
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
        cleanup {
            // Clean up workspace
            cleanWs()
        }
    }
}
