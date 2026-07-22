pipeline {
    agent any
    
    environment {
        // Targets the AWS Mumbai Region
        AWS_REGION       = 'ap-south-1' 
        FUNCTION_NAME    = 'mumbai-express-lambda'
        // Replace with your actual AWS IAM Role ARN
        IAM_ROLE_ARN     = 'arn:aws:iam::123456789012:role/lambda-mumbai-execution-role' 
        // Name of the credentials configured in Jenkins Credentials Provider
        AWS_CREDS_ID     = 'aws-mumbai-credentials' 
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                // Pulls your application code from GitHub/GitLab
                checkout scm
            }
        }
        
        stage('Package Application') {
            steps {
                echo 'Packaging Lambda code into a deployment zip file...'
                // Zips the workspace directory contents for deployment
                sh 'zip -r lambda_function.zip .'
            }
        }
        
        stage('Deploy to AWS Mumbai') {
            steps {
                // Wrapper to inject your AWS Access Keys safely
                withAWS(region: "${AWS_REGION}", credentials: "${AWS_CREDS_ID}") {
                    script {
                        echo "Checking if Lambda function '${FUNCTION_NAME}' exists in ap-south-1..."
                        
                        // Check function status without failing the build if it doesn't exist
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
