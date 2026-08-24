pipeline {
    agent any

    environment {
        // AWS S3 Storage Details
        S3_BUCKET    = 'code-version'
        PACKAGE_NAME = 'bookstore-package.zip'
        AWS_CREDS_ID = 'aws-credentials-id'  // Matches your Jenkins AWS credentials ID
        
        // Target Amazon Linux VM Configuration Details
        VM_IP        = '3.110.118.236'
        VM_USER      = 'ec2-user'      
        SSH_CREDS_ID = 'vm-ssh-key'    // Matches your Jenkins VM private key ID
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling the deployment pipeline configuration...'
                checkout scm
            }
        }

        stage('Secure S3 Fetch and Container Deployment') {
            steps {
                // 1. Pull both your secure AWS keys and your SSH key from the Jenkins vault safely
                withCredentials([usernamePassword(credentialsId: "${AWS_CREDS_ID}", 
                                                 usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                 passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    
                    sshagent(credentials: ["${SSH_CREDS_ID}"]) {
                        echo "Connecting securely to Amazon Linux VM: ${VM_IP}..."
                        
                        sh """
                            ssh -o StrictHostKeyChecking=no ${VM_USER}@${VM_IP} '
                                echo "Successfully logged in! Setting up the Docker engine..."
                                
                                # 2. Stop and remove the host-level native Nginx to completely free up Port 80
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                
                                # 3. Ensure Docker, unzip, and AWS CLI are fully installed on the machine
                                sudo yum install -y docker unzip awscli --skip-broken
                                
                                # 4. Make sure the Docker daemon background service is active and running
                                sudo systemctl enable --now docker
                                
                                # 5. Give your ec2-user account permissions to handle Docker commands
                                sudo usermod -aG docker ${VM_USER} || true
                                
                                # 6. Create an isolated workspace folder inside /tmp
                                cd /tmp
                                sudo rm -rf docker-container-deploy
                                mkdir -p docker-container-deploy
                                cd docker-container-deploy
                                
                                # 7. Pass your Jenkins AWS keys into the environment session memory
                                export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                                export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                                export AWS_DEFAULT_REGION=ap-south-1
                                
                                echo "Downloading your package securely via AWS CLI from bucket: ${S3_BUCKET}..."
                                aws s3 cp s3://${S3_BUCKET}/${PACKAGE_NAME} .
                                
                                # 8. Extract the zip file layers completely
                                echo "Extracting application assets..."
                                unzip -o ${PACKAGE_NAME}
                                
                                # 9. Handle nested sub-folders automatically if they exist
                                if [ ! -f "index.html" ]; then
                                    mv */index.html . 2>/dev/null || true
                                fi
                                
                                # 10. Create a fresh, dedicated Dockerfile to host your storefront code
                                echo "Generating custom Nginx container blueprint..."
                                cat <<EOF > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
                                
                                # 11. Stop and remove any old running bookstore container to avoid port conflicts
                                echo "Cleaning out stale containers..."
                                sudo docker stop bookstore-prod-site || true
                                sudo docker rm bookstore-prod-site || true
                                
                                # 12. Build your brand new custom isolated image
                                echo "Building fresh Docker image layer..."
                                sudo docker build -t bookstore-image:latest .
                                
                                # 13. Launch your live application inside a fresh container on Port 80
                                echo "Launching live bookstore container..."
                                sudo docker run -d -p 80:80 --name bookstore-prod-site --restart always bookstore-image:latest
                                
                                # 14. Clean up temporary directory paths from the host system
                                cd /tmp
                                rm -rf docker-container-deploy
                                
                                echo "Success! Your bookstore is running cleanly inside a Docker container."
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Finished Successfully! Your container is running live at http://3.110.118.236'
        }
        failure {
            echo 'Deployment Failed. Please check the console log outputs to troubleshoot the error.'
        }
    }
}
