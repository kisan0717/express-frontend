
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

                    test -f server.js
                    test -f package.json
                    test -f package-lock.json

                    rsync -av \
                        --exclude='node_modules/' \
                        --exclude='.git/' \
                        --exclude='.env' \
                        ./ "$DEPLOY_DIR/"

                    cd "$DEPLOY_DIR"
                    npm ci --omit=dev

                    pm2 describe express-frontend >/dev/null 2>&1 \
                        && pm2 restart express-frontend \
                        || pm2 start server.js --name express-frontend

                    pm2 save

                    curl --fail --silent --show-error \
                        http://127.0.0.1:3000/
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
