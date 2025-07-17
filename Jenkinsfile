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
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm ci
                    npm install serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test
                    kill $(jobs -p) || true
                '''
            }
                }
    }
    post {
        always {
            junit 'jest-results/junit.xml'
        }
    }
}
