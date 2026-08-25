pipeline {
    agent any

    environment {
        // AWS S3 Storage Details
        S3_BUCKET    = 'code-version'
        PACKAGE_NAME = 'bookstore-package.zip'
        AWS_CREDS_ID = 'aws-credentials-id'  // Matches your Jenkins AWS credentials ID
        
        // Target Amazon Linux VM Configuration Details
        VM_IP        = '43.204.219.68'
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
                                echo "Successfully logged in! Managing system services..."
                                
                                # 2. Stop host native Nginx to completely free up public Port 80
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                
                                # 3. Ensure Docker, unzip, and AWS CLI are fully installed on the machine
                                sudo yum install -y docker unzip awscli --skip-broken
                                
                                # 4. Make sure the Docker daemon background service is active and running
                                sudo systemctl enable --now docker
                                
                                # 5. Give your user profile permanent rights to manage containers
                                sudo usermod -aG docker ${VM_USER} || true
                                
                                # 6. Open Port 80 on the Amazon Linux internal OS firewall
                                sudo firewall-cmd --permanent --add-port=80/tcp || true
                                sudo firewall-cmd --reload || true
                                
                                # 7. Create an isolated workspace folder inside /tmp
                                cd /tmp
                                sudo rm -rf docker-reversed-deploy
                                mkdir -p docker-reversed-deploy
                                cd docker-reversed-deploy
                                
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
                                
                                # 11. Create a specialized Dockerfile that reconfigures Nginx to listen on Port 8090
                                echo "Generating custom Nginx blueprint listening on Port 8090..."
                                cat <<EOF > Dockerfile
FROM nginx:alpine
# Modify default Nginx config to listen on port 8090 instead of 80
RUN sed -i "s/listen       80;/listen       8090;/g" /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/
EXPOSE 8090
CMD ["nginx", "-g", "daemon off;"]
EOF
                                
                                # 12. Stop and remove any old running bookstore container to avoid conflicts
                                echo "Cleaning out stale containers..."
                                sudo docker stop bookstore-prod-site || true
                                sudo docker rm bookstore-prod-site || true
                                
                                # 13. Build your brand new custom isolated image
                                echo "Building fresh Docker image layer..."
                                sudo docker build -t bookstore-image:latest .
                                
                                # 14. Launch container mapping VM Host Port 80 to Internal Container Port 8090
                                echo "Launching live bookstore container (Host 80 -> Container 8090)..."
                                sudo docker run -d -p 80:8090 --name bookstore-prod-site --restart always bookstore-image:latest
                                
                                # 15. Clean up temporary directory paths from the host system
                                cd /tmp
                                rm -rf docker-reversed-deploy
                                
                                echo "Checking localized socket response on VM port 80..."
                                curl -I http://localhost:80
                                
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
            echo "Pipeline Finished Successfully! Your container is running live at http://3.110.118.236"
        }
        failure {
            echo 'Deployment Failed. Please check the console log outputs to troubleshoot the error.'
        }
    }
}
