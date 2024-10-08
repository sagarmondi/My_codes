pipeline {
    agent any
	environment {
        PRODUCTION_IP_ADDRESS = '35.162.27.193'
        
    }
	
    
    tools {
        nodejs "nodejs"
    }

    stages {
        stage('Install Packages') {
            steps {
                script {
                    sh 'yarn install'
                }
            }
        }

        stage('Run the App') {
            steps {
                script {
                    sh 'yarn start:pm2'
                    sleep 5
                }
            }
        }

        stage('Test the app') {
            steps {
                script {
                    sh 'curl http://localhost:3000/health'
                }
            }
        }

        stage('Stop the App') {
            steps {
                script {
                    sh 'pm2 stop todos-app'
                }
            }
        }  
		 stage('Debug Environment Variable') {
            steps {
                script {
                    echo "PRODUCTION_IP_ADDRESS: $PRODUCTION_IP_ADDRESS"
                }
            }
        }
		

        stage('Add Host to known_hosts') {
            steps {
                script {
                    sh '''
                        mkdir -p /var/lib/jenkins/.ssh
                        ssh-keyscan -H $PRODUCTION_IP_ADDRESS >> /var/lib/jenkins/.ssh/known_hosts
                    '''
                }
            }
        }

        stage('Deploy') {
            environment {
                DEPLOY_SSH_KEY = credentials('AWS_INSTANCE_SSH')
            }

            steps {
                script {
                    sh '''
                        chmod 600 $DEPLOY_SSH_KEY
                        ssh -v -i $DEPLOY_SSH_KEY ubuntu@$PRODUCTION_IP_ADDRESS '
                            if [ ! -d "todos-app" ]; then
                                git clone https://github.com/sagarmondi/My_codes.git My_codes
                                cd My_codes
                            else
                                cd My_codes
                                git pull
                            fi

                            yarn install

                            if pm2 describe My_codes > /dev/null ; then
                                pm2 restart My_codes
                            else
                                yarn start:pm2
                            fi
                        '
                    '''
                }
            }
        }
    }
}
