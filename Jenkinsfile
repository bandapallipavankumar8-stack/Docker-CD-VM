pipeline {
    agent any

    environment {
        S3_BUCKET    = 'code-version'
        PACKAGE_NAME = 'bookstore-package.zip'
        AWS_CREDS_ID = 'aws-credentials-id'
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
                        
                        sh '''
                            ssh -o StrictHostKeyChecking=no ${VM_USER}@${VM_IP} "
                                echo 'Logged in successfully! Cleaning up old native services...'
                                
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                sudo yum install -y docker unzip awscli --skip-broken
                                sudo systemctl enable --now docker
                                sudo usermod -aG docker ${VM_USER} || true
                                
                                cd /tmp
                                sudo rm -rf docker-reversed-deploy
                                mkdir -p docker-reversed-deploy
                                cd docker-reversed-deploy
                                
                                export AWS_ACCESS_KEY_ID='${AWS_ACCESS_KEY_ID}'
                                export AWS_SECRET_ACCESS_KEY='${AWS_SECRET_ACCESS_KEY}'
                                export AWS_DEFAULT_REGION='ap-south-1'
                                
                                echo 'Downloading package from S3...'
                                aws s3 cp s3://${S3_BUCKET}/${PACKAGE_NAME} .
                                
                                echo 'Extracting application assets...'
                                unzip -o ${PACKAGE_NAME}
                                
                                if [ ! -f 'index.html' ]; then
                                    mv */index.html . 2>/dev/null || true
                                fi
                                
                                # FIX: Simplified plain-text layout to eliminate formatting crashes
                                echo 'Generating streamlined blueprint...'
                                echo 'FROM nginx:alpine' > Dockerfile
                                echo 'RUN sed -i \"s/listen       80;/listen       8090;/g\" /etc/nginx/conf.d/default.conf' >> Dockerfile
                                echo 'COPY index.html /usr/share/nginx/html/' >> Dockerfile
                                echo 'EXPOSE 8090' >> Dockerfile
                                echo 'CMD nginx -g \"daemon off;\"' >> Dockerfile
                                
                                echo 'Managing container states...'
                                docker stop bookstore-prod-site || true
                                docker rm bookstore-prod-site || true
                                
                                echo 'Building fresh Docker image layer...'
                                docker build -t bookstore-image:latest .
                                
                                echo 'Launching live bookstore container...'
                                docker run -d -p 80:8090 --name bookstore-prod-site --restart always bookstore-image:latest
                                
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
