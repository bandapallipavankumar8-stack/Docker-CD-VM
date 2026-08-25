pipeline {
    agent any

    environment {
        S3_BUCKET    = 'code-version'
        PACKAGE_NAME = 'bookstore-package.zip'
        AWS_CREDS_ID = 'aws-credentials-id'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling the latest code from your GitHub repository...'
                checkout scm
            }
        }

        stage('Package Application') {
            steps {
                echo 'Compressing files into a deployment package...'
                sh "zip -r ${PACKAGE_NAME} Dockerfile index.html"
            }
        }

        stage('Upload to AWS S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${AWS_CREDS_ID}", 
                                                 usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                 passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    echo "Uploading ${PACKAGE_NAME} to S3 bucket: ${S3_BUCKET}..."
                    sh "aws s3 cp ${PACKAGE_NAME} s3://${S3_BUCKET}/"
                }
            }
        }

        // ============================================================
        // ADD THIS NEW STAGE TO LINK CI TO YOUR CD PIPELINE
        // ============================================================
        stage('Trigger CD Pipeline') {
            steps {
                echo 'CI complete! Triggering your CD deployment job automatically...'
                
                // Replace 'My-CD-Deployment-Job' with the exact name of your CD job in Jenkins
                build job: 'My-CD-Deployment-Job', wait: false
            }
        }
    }

    post {
        always {
            echo 'Cleaning up the build environment...'
            sh "rm -f ${PACKAGE_NAME}"
        }
        success {
            echo 'CI Pipeline Success! S3 Upload completed and CD triggered.'
        }
        failure {
            echo 'CI Pipeline Failed. CD was not triggered.'
        }
    }
}
