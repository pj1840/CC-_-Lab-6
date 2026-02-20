pipeline {
    agent any

    stages {
        stage('Build Backend Image') {
            steps {
                sh '''
                set -eux

                # Ensure base image exists locally (avoids registry auth EOF during build)
                docker pull ubuntu:22.04 || true

                # Disable BuildKit to avoid "load metadata/auth token" failures
                export DOCKER_BUILDKIT=0

                docker rmi -f backend-app 2>/dev/null || true
                docker build -t backend-app backend
                '''
            }
        }

        stage('Deploy Backend Containers') {
            steps {
                sh '''
                set -eux

                docker network create app-network 2>/dev/null || true
                docker rm -f backend1 backend2 2>/dev/null || true

                docker run -d --name backend1 --network app-network backend-app
                docker run -d --name backend2 --network app-network backend-app

                # small delay so containers are ready before nginx tries to connect
                sleep 2
                '''
            }
        }

        stage('Deploy NGINX Load Balancer') {
            steps {
                sh '''
                set -eux

                docker pull nginx:latest || true
                docker rm -f nginx-lb 2>/dev/null || true

                docker run -d \
                  --name nginx-lb \
                  --network app-network \
                  -p 80:80 \
                  nginx:latest

                sleep 2

                docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
                docker exec nginx-lb nginx -t
                docker exec nginx-lb nginx -s reload

                docker ps --filter "name=backend"
                docker ps --filter "name=nginx-lb"
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully. NGINX load balancer is running.'
        }
        failure {
            echo 'Pipeline failed. Check console logs for errors.'
        }
    }
}