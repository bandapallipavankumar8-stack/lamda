pipeline {
    agent any
    
    environment {
        // Define your deployment configurations
        AWS_REGION      = 'ap-south-1' // Mumbai Region
        LAMBDA_NAME     = 'my-lambda-function'
        LAMBDA_HANDLER  = 'index.handler'  // e.g., filename.methodName
        LAMBDA_RUNTIME  = 'nodejs18.x'    // Change based on your programming language
        LAMBDA_ROLE_ARN = 'arn:aws:iam::123456789012:role/service-role/your-lambda-execution-role' // Replace with a basic Lambda execution role ARN
        CREDENTIALS_ID  = 'aws-admin-creds' // Must match the ID configured in Jenkins Step 1
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Pulls the main branch from your SCM
                checkout scm
            }
        }

        stage('Package Application') {
            steps {
                echo 'Packaging application files into ZIP configuration...'
                // Excludes Jenkins configuration and hidden Git assets so zip error is avoided
                sh 'zip -r lambda_function.zip . -x "*.git*" -x "Jenkinsfile"'
            }
        }

        stage('Deploy to AWS Mumbai') {
            steps {
                // Securely injects your Jenkins admin credentials
                withAWS(credentials: "${CREDENTIALS_ID}", region: "${AWS_REGION}") {
                    script {
                        echo 'Checking if Lambda function already exists...'
                        def exists = sh(script: "aws lambda get-function --function-name ${LAMBDA_NAME} 2>&1", returnStatus: true)
                        
                        if (exists == 0) {
                            echo "Lambda function exists. Updating code deployment..."
                            sh """
                                aws lambda update-function-code \
                                    --function-name ${LAMBDA_NAME} \
                                    --zip-file fileb://lambda_function.zip
                            """
                        } else {
                            echo "Lambda function does not exist. Creating new function..."
                            sh """
                                aws lambda create-function \
                                    --function-name ${LAMBDA_NAME} \
                                    --runtime ${LAMBDA_RUNTIME} \
                                    --role ${LAMBDA_ROLE_ARN} \
                                    --handler ${LAMBDA_HANDLER} \
                                    --zip-file fileb://lambda_function.zip
                            """
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Check AWS configurations, workspace files, or your Jenkins AWS credential settings.'
        }
    }
}
