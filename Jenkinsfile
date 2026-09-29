pipeline {
    agent any

    environment {
        COMPOSE_FILE = 'docker-compose.yml'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out Contact Management application'
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "Docker:"
                    docker --version

                    echo "Docker Compose:"
                    docker compose version

                    echo "Git:"
                    git --version

                    echo "Java:"
                    java -version
                '''
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                sh '''
                    echo "Installing frontend dependencies..."

                    npm install

                    echo "Installing React TypeScript definitions..."

                    npm install --save-dev @types/react @types/react-dom
                '''
            }
        }

        stage('Frontend Build Test') {
            steps {
                sh '''
                    echo "Building React frontend..."

                    npm run build
                '''
            }
        }

        stage('Create Docker Files') {
            steps {
                sh '''
                    echo "Creating frontend Dockerfile..."

                    cat > Dockerfile <<'EOF'
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm install

RUN npm install --save-dev @types/react @types/react-dom

COPY . .

RUN npm run build


FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
EOF


                    echo "Creating backend Dockerfile..."

                    cat > backend/Dockerfile <<'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF


                    echo "Creating nginx configuration..."

                    cat > nginx.conf <<'EOF'
server {
    listen 80;

    server_name _;

    root /usr/share/nginx/html;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF


                    echo "Creating Docker Compose file..."

                    cat > docker-compose.yml <<'EOF'
services:

  db:
    image: postgres:16-alpine
    container_name: contact-management-db
    environment:
      POSTGRES_USER: contactuser
      POSTGRES_PASSWORD: contactpassword
      POSTGRES_DB: contactdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - contact-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U contactuser -d contactdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: contact-management-backend
    environment:
      DATABASE_URL: postgresql://contactuser:contactpassword@db:5432/contactdb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - contact-network
    ports:
      - "8000:8000"

  frontend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: contact-management-frontend
    depends_on:
      - backend
    ports:
      - "8081:80"
    networks:
      - contact-network

networks:
  contact-network:
    driver: bridge

volumes:
  postgres_data:
EOF

                    echo "Docker files created successfully."

                    ls -la
                '''
            }
        }

        stage('Validate Docker Compose') {
            steps {
                sh '''
                    echo "Validating Docker Compose..."

                    test -f docker-compose.yml

                    docker compose -f docker-compose.yml config
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    echo "Building Docker images..."

                    docker compose -f docker-compose.yml build --no-cache
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "Stopping previous containers..."

                    docker compose -f docker-compose.yml down || true

                    echo "Starting application..."

                    docker compose -f docker-compose.yml up -d
                '''
            }
        }

        stage('Verify Containers') {
            steps {
                sh '''
                    echo "Checking containers..."

                    docker compose -f docker-compose.yml ps
                '''
            }
        }

        stage('Backend Health Check') {
            steps {
                sh '''
                    echo "Checking backend..."

                    sleep 10

                    docker compose -f docker-compose.yml logs --tail=50 backend

                    curl -f http://localhost:8000/docs || exit 1
                '''
            }
        }

        stage('Application Test') {
            steps {
                sh '''
                    echo "Testing frontend..."

                    curl -f http://localhost:8081 || exit 1

                    echo "Application is running successfully."
                '''
            }
        }
    }

    post {

        success {
            echo '=========================================='
            echo 'CONTACT MANAGEMENT DEPLOYMENT SUCCESSFUL'
            echo '=========================================='
            echo 'Frontend: http://EC2-PUBLIC-IP:8081'
            echo 'Backend:  http://EC2-PUBLIC-IP:8000/docs'
        }

        failure {
            echo '=========================================='
            echo 'CONTACT MANAGEMENT DEPLOYMENT FAILED'
            echo '=========================================='

            sh '''
                docker compose -f docker-compose.yml ps || true
                docker compose -f docker-compose.yml logs --tail=100 || true
            '''
        }

        always {
            echo 'Pipeline completed.'
        }
    }
}
