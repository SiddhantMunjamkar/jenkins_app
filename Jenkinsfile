pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    test -f build/index.html
                    if [ $? -eq 0 ]; then
                      echo "index.html exists. Running tests..."
                      npm run test
                    else
                      echo "index.html does not exist!"
                      exit 1
                    fi
                '''
            }
        }
        stage('E2E Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm ci
                    npm install serve
                    node_modules/.bin/serve -s build &
                    server_pid=$!
                    sleep 10
                    PLAYWRIGHT_JUNIT_OUTPUT_NAME=test-results/test-results.xml npx playwright test --reporter=junit
                    kill $server_pid || true
                '''
            }
                }
    }
    post {
        always {
            junit(
                testResults: ['jest-results/junit.xml', 'test-results/test-results.xml'],
                allowEmptyResults: true
            )
        }
    }
}
