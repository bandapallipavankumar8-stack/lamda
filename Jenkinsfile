pipeline {
    agent any
    
    environment {
        AWS_REGION      = 'ap-south-1'               // Mumbai Region
        LAMBDA_NAME     = 'my-jenkins-lambda'        // Name of your Lambda function
        LAMBDA_RUNTIME  = 'nodejs18.x'              // Node.js runtime environment
        ROLE_NAME       = 'jenkins-lambda-exec-role' // IAM Role name to be generated/used
        CREDENTIALS_ID  = 'aws-admin-creds'          // The AWS credentials ID set up in Jenkins
        
        // =========================================================================
        // UPDATE THESE TWO VALUES TO MATCH YOUR SEPARATE FOLDER STRUCTURE
        // =========================================================================
        CODE_FOLDER     = 'src'                      // Change 'src' to the exact name of your separate folder
        LAMBDA_HANDLER  = 'src/index.handler'        // Format: folderName/fileName.methodName
        // =========================================================================
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Package Application') {
            steps {
                echo "Packaging application code from the '${CODE_FOLDER}' folder into ZIP file..."
                script {
                    // Navigate into your separate folder, zip all its contents, and place the zip file in the root workspace
                    sh "cd ${CODE_FOLDER} && zip -r ../lambda_function.zip . -x '*.git*'"
                }
            }
        }

        stage('Deploy to AWS Mumbai') {
            steps {
                withAWS(credentials: "${CREDENTIALS_ID}", region: "${AWS_REGION}") {
                    script {
                        // Step 1: Ensure an Execution Role exists for the Lambda function
                        echo 'Verifying IAM execution role...'
                        def roleCheck = sh(script: "aws iam get-role --role-name ${ROLE_NAME} 2>&1", returnStatus: true)
                        def roleArn = ""

                        if (roleCheck != 0) {
                            echo "IAM Role does not exist. Creating ${ROLE_NAME}..."
                            def trustPolicy = '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"://amazonaws.com"},"Action":"sts:AssumeRole"}]}'
                            
                            sh "aws iam create-role --role-name ${ROLE_NAME} --assume-role-policy-document '${trustPolicy}'"
                            sh "aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
                            
                            echo "Waiting for IAM Role propagation..."
                            sleep(time: 10, unit: 'SECONDS')
                        }
                        
                        roleArn = sh(script: "aws iam get-role --role-name ${ROLE_NAME} --query 'Role.Arn' --output text", returnStdout: true).trim()
                        echo "Using Role ARN: ${roleArn}"

                        // Step 2: Deploy or update the Lambda function
                        echo 'Checking if Lambda function already exists...'
                        def lambdaCheck = sh(script: "aws lambda get-function --function-name ${LAMBDA_NAME} 2>&1", returnStatus: true)
                        
                        if (lambdaCheck == 0) {
                            echo "Lambda function exists. Updating live deployment package..."
                            sh """
                                aws lambda update-function-code \
                                    --function-name ${LAMBDA_NAME} \
                                    --zip-file fileb://lambda_function.zip
                            """
                        } else {
                            echo "Lambda function does not exist. Provisioning new function..."
                            sh """
                                aws lambda create-function \
                                    --function-name ${LAMBDA_NAME} \
                                    --runtime ${LAMBDA_RUNTIME} \
                                    --role ${roleArn} \
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
            echo 'Pipeline completed successfully. Code deployed to AWS Mumbai!'
        }
        failure {
            echo 'Pipeline failed. Check folder paths, zip utilities, or AWS console permissions.'
        }
    }
}
