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
        
        // Custom Target Browser Port
        TARGET_PORT  = '8090'
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
                                echo "Successfully logged in! Managing system services..."
                                
                                # 2. Stop host native Nginx to prevent any underlying networking conflicts
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                
                                # 3. Ensure Docker, unzip, and AWS CLI are fully installed on the machine
                                sudo yum install -y docker unzip awscli --skip-broken
                                
                                # 4. Make sure the Docker daemon background service is active and running
                                sudo systemctl enable --now docker
                                
                                # 5. Give your user profile permanent rights to manage containers
                                sudo usermod -aG docker ${VM_USER} || true
                                
                                # 6. AGGRESSIVE FIREWALL FIX: Force-open port 8090 on the inner OS layer
                                echo "Unblocking network channels for port ${TARGET_PORT}..."
                                sudo systemctl start firewalld || true
                                sudo firewall-cmd --permanent --add-port=${TARGET_PORT}/tcp || true
                                sudo firewall-cmd --reload || true
                                
                                # 7. Create an isolated workspace folder inside /tmp
                                cd /tmp
                                sudo rm -rf docker-final-deploy
                                mkdir -p docker-final-deploy
                                cd docker-final-deploy
                                
                                # 8. Pass your Jenkins AWS keys into the environment session memory
                                export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                                export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                                export AWS_DEFAULT_REGION=ap-south-1
                                
                                echo "Downloading package securely via AWS CLI from bucket: ${S3_BUCKET}..."
                                aws s3 cp s3://${S3_BUCKET}/${PACKAGE_NAME} .
                                
                                # 9. Extract the zip file layers completely
                                echo "Extracting application assets..."
                                unzip -o ${PACKAGE_NAME}
                                
                                # 10. Handle nested sub-folders automatically if they exist
                                if [ ! -f "index.html" ]; then
                                    mv */index.html . 2>/dev/null || true
                                fi
                                
                                # 11. Create a fresh, dedicated Dockerfile to host your storefront code
                                echo "Generating custom Nginx container blueprint..."
                                cat <<EOF > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
                                
                                # 12. Stop and remove any old running bookstore container to avoid port conflicts
                                echo "Cleaning out stale containers..."
                                sudo docker stop bookstore-prod-site || true
                                sudo docker rm bookstore-prod-site || true
                                
                                # 13. Build your brand new custom isolated image
                                echo "Building fresh Docker image layer..."
                                sudo docker build -t bookstore-image:latest .
                                
                                # 14. Launch your live container mapping host port 8090 to internal container port 80
                                echo "Launching live bookstore container container app on port ${TARGET_PORT}..."
                                sudo docker run -d -p ${TARGET_PORT}:80 --name bookstore-prod-site --restart always bookstore-image:latest
                                
                                # 15. Clean up temporary directory paths from the host system
                                cd /tmp
                                rm -rf docker-final-deploy
                                
                                echo "Checking localized socket response on port ${TARGET_PORT}..."
                                curl -I http://localhost:${TARGET_PORT}
                                
                                echo "Success! Your container deployment is complete."
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline Finished Successfully! Your container is running live at http://3.110.118.236:${TARGET_PORT}"
        }
        failure {
            echo 'Deployment Failed. Please check the console log outputs to troubleshoot the error.'
        }
    }
}
