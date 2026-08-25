pipeline {
    agent any

    stages {
        stage('CD: Checkout Config') {
            steps {
                echo 'Pulling the deployment pipeline configuration...'
                checkout scm
            }
        }

        stage('CD: Fetch and Deploy Docker Container') {
            steps {
                // Uses standard Jenkins Secret Text strings to bypass missing AWS plugin
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sshagent(['ec2-user']) {
                        echo 'Connecting securely to Amazon Linux VM: 43.204.219.68...'
                        sh '''
                            ssh -o StrictHostKeyChecking=no ec2-user@43.204.219.68 '
                                echo "Logged in successfully! Cleaning up old native services..."
                                
                                # Stop native Nginx if running to free up port 80
                                sudo systemctl stop nginx || true
                                sudo systemctl disable nginx || true
                                
                                # Install core runtime dependencies if missing
                                sudo yum install -y docker unzip awscli --skip-broken
                                sudo systemctl enable --now docker
                                sudo usermod -aG docker ec2-user || true
                                
                                # Setup isolated workspace path
                                cd /tmp
                                sudo rm -rf docker-reversed-deploy
                                mkdir -p docker-reversed-deploy
                                cd docker-reversed-deploy
                                
                                # Mount keys dynamically inside session terminal memory
                                export AWS_ACCESS_KEY_ID="'$AWS_ACCESS_KEY_ID'"
                                export AWS_SECRET_ACCESS_KEY="'$AWS_SECRET_ACCESS_KEY'"
                                export AWS_DEFAULT_REGION="ap-south-1"
                                
                                echo "Downloading package from S3..."
                                aws s3 cp s3://code-version/bookstore-package.zip .
                                
                                echo "Extracting application assets..."
                                unzip -o bookstore-package.zip
                                
                                # Fallback handling for pathing structures
                                if [ ! -f "index.html" ]; then
                                    mv */index.html . 2>/dev/null || true
                                fi
                                
                                echo "Managing container states..."
                                docker stop bookstore-prod-site || true
                                docker rm bookstore-prod-site || true
                                
                                echo "Building fresh Docker image layer..."
                                docker build --no-cache -t bookstore-image:latest .
                                
                                echo "Launching live bookstore container on host port 80..."
                                docker run -d -p 80:80 --name bookstore-prod-site --restart always bookstore-image:latest > /dev/null 2>&1 &
                                
                                # Give the container a moment to spin up before verifying
                                sleep 5
                                
                                echo "Verifying local web response on port 80..."
                                curl -I http://localhost:80
                                
                                # Clear workspace cache
                                cd /tmp
                                rm -rf docker-reversed-deploy
                            '
                        '''
                    }
                }
            }
        }
    }
}
