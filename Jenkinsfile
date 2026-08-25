pipeline {
    agent any

    environment {
        // AWS S3 Storage Details
        S3_BUCKET    = 'code-version'
        PACKAGE_NAME = 'bookstore-package.zip'
        AWS_CREDS_ID = 'aws-credentials-id'
        
        // Target Amazon Linux VM Configurations
        VM_IP        = '43.204.219.68'
        VM_USER      = 'ec2-user'      
        SSH_CREDS_ID = 'vm-ssh-key' 
    }

    stages {
        stage('CD: Checkout Config') {
            steps {
                echo 'Pulling the deployment pipeline configuration...'
                checkout scm
            }
        }

        stage('CD: Fetch and Deploy Docker Container') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${AWS_CREDS_ID}", 
                                                 usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                 passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    
                    sshagent(credentials: ["${SSH_CREDS_ID}"]) {
                        echo "Connecting securely to Amazon Linux VM: ${VM_IP}..."
                        
                        // CHANGED TO LITERAL SINGLE QUOTES FOR SHELL SAFETY
                        sh '''
                            ssh -o StrictHostKeyChecking=no ${VM_USER}@${VM_IP} "
                                echo 'Logged in successfully! Cleaning up old native services...'
                                
                                # Stop host native Nginx to completely free up public Port 80
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                
                                # Install core requirements if missing
                                sudo yum install -y docker unzip awscli --skip-broken
                                
                                # Start and enable Docker engine
                                sudo systemctl enable --now docker
                                sudo usermod -aG docker ${VM_USER} || true
                                
                                # Create isolated temporary deployment directory
                                cd /tmp
                                sudo rm -rf docker-reversed-deploy
                                mkdir -p docker-reversed-deploy
                                cd docker-reversed-deploy
                                
                                # Pass session credentials safely via shell memory
                                export AWS_ACCESS_KEY_ID='${AWS_ACCESS_KEY_ID}'
                                export AWS_SECRET_ACCESS_KEY='${AWS_SECRET_ACCESS_KEY}'
                                export AWS_DEFAULT_REGION='ap-south-1'
                                
                                echo 'Downloading package from S3...'
                                aws s3 cp s3://${S3_BUCKET}/${PACKAGE_NAME} .
                                
                                echo 'Extracting application assets...'
                                unzip -o ${PACKAGE_NAME}
                                
                                # Handle nested sub-folders automatically if they exist
                                if [ ! -f 'index.html' ]; then
                                    mv */index.html . 2>/dev/null || true
                                fi
                                
                                # Generate custom Nginx config blueprint listening internally on Port 8090
                                cat <<EOF > Dockerfile
FROM nginx:alpine
RUN sed -i 's/listen       80;/listen       8090;/g' /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/
EXPOSE 8090
CMD [\\"nginx\\", \\"-g\\", \\"daemon off;\\"]
EOF
                                
                                # Remove any stale running bookstore containers to avoid conflicts
                                echo 'Cleaning out old container builds...'
                                sudo docker stop bookstore-prod-site || true
                                sudo docker rm bookstore-prod-site || true
                                
                                # Build fresh custom image and run container mapping VM Port 80 to 8090
                                echo 'Building fresh Docker image layer...'
                                sudo docker build -t bookstore-image:latest .
                                
                                echo 'Launching live bookstore container...'
                                sudo docker run -d -p 80:8090 --name bookstore-prod-site --restart always bookstore-image:latest
                                
                                # Clean up the temp file workspace
                                cd /tmp
                                rm -rf docker-reversed-deploy
                                
                                echo 'Verifying local web response on port 80...'
                                curl -I http://localhost:80
                            "
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline Finished Successfully! Your container is running live at http://43.204.219.68"
        }
        failure {
            echo 'Deployment Failed. Please check the console log outputs to troubleshoot the error.'
        }
    }
}
