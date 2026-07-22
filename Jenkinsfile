pipeline {
    agent any
    
    environment {
        AWS_REGION       = 'ap-south-1' 
        FUNCTION_NAME    = 'mumbai-express-lambda'
        IAM_ROLE_ARN     = 'arn:aws:iam::123456789012:role/lambda-mumbai-execution-role' 
        AWS_CREDS_ID     = 'aws-mumbai-credentials' 
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        
        stage('Package Application') {
            steps {
                echo 'Packaging Lambda code (excluding git history)...'
                // Clean fix: Excludes bulky git configuration files from your lambda package
                sh 'zip -r lambda_function.zip . -x "*.git*" -x "Jenkinsfile"'
            }
        }
        
        stage('Deploy to AWS Mumbai') {
            steps {
                // Native block that works out of the box on all Jenkins instances
                withCredentials([usernamePassword(credentialsId: "${AWS_CREDS_ID}", 
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY', 
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    script {
                        // Injects credentials into your environment variables automatically
                        env.AWS_DEFAULT_REGION = "${AWS_REGION}"
                        
                        echo "Checking if Lambda function '${FUNCTION_NAME}' exists in ${AWS_REGION}..."
                        
                        def status = sh(
                            script: "aws lambda get-function --function-name ${FUNCTION_NAME} --region ${AWS_REGION}", 
                            returnStatus: true
                        )
                        
                        if (status != 0) {
                            echo "Function not found. Creating a new Lambda function in Mumbai..."
                            sh """
                            aws lambda create-function \
                                --region ${AWS_REGION} \
                                --function-name ${FUNCTION_NAME} \
                                --runtime python3.12 \
                                --role ${IAM_ROLE_ARN} \
                                --handler index.handler \
                                --zip-file fileb://lambda_function.zip
                            """
                        } else {
                            echo "Function found. Updating code for existing Lambda in Mumbai..."
                            sh """
                            aws lambda update-function-code \
                                --region ${AWS_REGION} \
                                --function-name ${FUNCTION_NAME} \
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
            echo "Successfully deployed Lambda to AWS Mumbai (ap-south-1)!"
        }
        failure {
            echo "Deployment failed. Check AWS IAM permissions or credentials."
        }
    }
}
