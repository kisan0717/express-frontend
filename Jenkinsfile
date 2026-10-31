pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DEPLOY_DIR = '/opt/apps/express-frontend'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -eu

                    npm ci

                    if node -e "
                        const p = require('./package.json');
                        process.exit(p.scripts && p.scripts.test ? 0 : 1)
                    "; then
                        npm test
                    else
                        echo "No test script configured; skipping tests."
                    fi
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -eu

                    echo "Validating application files..."

                    test -f server.js
                    test -f package.json
                    test -f package-lock.json
                    test -f public/index.html

                    echo "Copying application files to EC2..."

                    rsync -r \
                        --no-perms \
                        --no-owner \
                        --no-group \
                        --exclude='node_modules/' \
                        --exclude='.git/' \
                        --exclude='.env' \
                        ./ "$DEPLOY_DIR/"

                    cd "$DEPLOY_DIR"

                    echo "Installing production dependencies..."

                    npm ci --omit=dev

                    echo "Restarting Express using Ubuntu PM2..."

                    sudo -u ubuntu /usr/bin/pm2 restart express-frontend

                    echo "Checking Express health..."

                    curl --fail --silent --show-error \
                        http://127.0.0.1:3000/health

                    echo "Checking HTML frontend..."

                    curl --fail --silent --show-error \
                        http://127.0.0.1:3000/ \
                        | grep -qi '<!DOCTYPE html'

                    echo "Express deployment and HTML verification successful."
                '''
            }
        }
    }

    post {
        success {
            echo 'Express deployment completed successfully.'
        }

        failure {
            echo 'Express deployment failed. Check Console Output.'
        }
    }
}
