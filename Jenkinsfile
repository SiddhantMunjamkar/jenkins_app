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
    }
    post {
        always {
            junit 'test-results/junit.xml'
        }
    }
}
