pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '123456789012' // Replace with your AWS account ID
        APP_NAME = 'react-app'
        GIT_REPO_URL = 'https://github.com/your-org/react-app.git' // Replace with your repo URL
        
        // ECR repositories for different stages
        DEV_ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}-dev"
        STAGING_ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}-staging"
        PROD_ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}-prod"
        
        // Deploy stage is determined by a parameter
        DEPLOY_ENV = "${params.ENVIRONMENT ?: 'dev'}" // Default to dev if not specified
    }
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Select deployment environment')
    }

    stages {
        stage('Setup ECR Repositories') {
            steps {
                script {
                    // Create ECR repositories if they don't exist
                    sh '''
                        aws ecr describe-repositories --repository-names ${APP_NAME}-dev || \
                        aws ecr create-repository --repository-name ${APP_NAME}-dev
                        
                        aws ecr describe-repositories --repository-names ${APP_NAME}-staging || \
                        aws ecr create-repository --repository-name ${APP_NAME}-staging
                        
                        aws ecr describe-repositories --repository-names ${APP_NAME}-prod || \
                        aws ecr create-repository --repository-name ${APP_NAME}-prod
                        
                        # ECR login
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }
        
        stage('Clone and Setup') {
            steps {
                // Clean workspace and clone repository
                cleanWs()
                git branch: 'main', url: "${GIT_REPO_URL}"
                
                // Change into the cloned directory and install Node.js dependencies
                sh '''
                    cd ${WORKSPACE}
                    npm install
                    npm run build
                '''
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    def ecrRepo
                    
                    // Select ECR repository based on environment
                    switch(DEPLOY_ENV) {
                        case 'dev':
                            ecrRepo = DEV_ECR_REPO
                            break
                        case 'staging':
                            ecrRepo = STAGING_ECR_REPO
                            break
                        case 'prod':
                            ecrRepo = PROD_ECR_REPO
                            break
                    }
                    
                    // Build Docker image
                    sh """
                        docker build -t ${ecrRepo}:${imageTag} -t ${ecrRepo}:latest .
                        docker push ${ecrRepo}:${imageTag}
                        docker push ${ecrRepo}:latest
                    """
                }
            }
        }
    }
    
    post {
        always {
            // Clean up Docker images
            sh 'docker system prune -f'
        }
    }
} 