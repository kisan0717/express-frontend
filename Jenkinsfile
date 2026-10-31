
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

                    echo "Installing dependencies..."

                    npm ci

                    echo "Running Express tests..."

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

                    echo "Waiting for Express to become healthy..."

                    for i in $(seq 1 15); do

                        if curl --fail --silent \
                            http://127.0.0.1:3000/health >/dev/null; then

                            echo "Express health check passed."
                            break
                        fi

                        if [ "$i" -eq 15 ]; then
                            echo "ERROR: Express failed to start."

                            sudo -u ubuntu /usr/bin/pm2 logs express-frontend --lines 50

                            exit 1
                        fi

                        echo "Waiting for Express... attempt $i"
                        sleep 2
                    done

                    echo "Checking HTML frontend..."

                    curl --fail --silent --show-error \
                        http://127.0.0.1:3000/ \
                        | grep -qi '<!DOCTYPE html'

                    echo "HTML frontend verified successfully."

                    echo "Checking PM2 status..."

                    echo "PM2 restart completed successfully."

                    echo "Express deployment completed successfully."
                '''
            }
        }
    }

    post {

        success {
            echo 'SUCCESS: Express frontend deployed successfully.'
        }

        failure {
            echo 'FAILURE: Express deployment failed. Check Console Output.'
        }

        always {
            echo 'Express CI/CD pipeline execution completed.'
        }
    }
}
