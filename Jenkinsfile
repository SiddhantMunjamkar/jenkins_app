pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '5d553fe2-c386-415a-88bc-72674a13bc6d'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token') // Ensure you have this credential set up in Jenkins
    }
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
                    echo "Building the application..."
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Build docker image') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                    args " -u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            steps {
                sh '''
                    amazon-linux-extras install docker
                    docker build -t myjenkinsapp .
                '''
            }
        }
        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args " -u root --entrypoint=''"
                    reuseNode true
                }
            }
            environment {
                REACT_APP_VERSION = "1.0.$BUILD_ID"
                AWS_S3_BUCKET = 'jenkins-learn-siddhant'
                AWS_DEFAULT_REGION = 'ap-south-1'
                AWS_ECS_CLUSTER = 'jenkins-learn-siddhant-cluster'
                AWS_ECS_SERVICE = 'jenkins-learn-siddhant-service'
                AWS_ECS_TASK_DEFINITION = 'learn-jenkins-app-prod'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    // some block
                    // echo "Configuring AWS CLI..." > index.html
                    // aws s3 cp  index.html s3://$AWS_S3_BUCKET/index.html
                    // aws --version
                    // aws s3 sync build s3://$AWS_S3_BUCKET
                    sh '''
                        aws --version
                        yum install jq -y
                        LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revision')
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER_ --service  --task-definition  $AWS_ECS_TASK_DEFINITION:$LATEST_TD_REVISION
                        aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICE jenkins-learn-siddhant-service
                    '''
                }
            }
        }

        stage('Unit tests') {
            parallel {
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
                                echo "index.html exists. Running unit tests..."
                                npm run test -- --watchAll=false --forceExit
                            else
                                echo "index.html does not exist!"
                                exit 1
                            fi
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
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
                            npm install serve
                            node_modules/.bin/serve -s build &  # Start static server
                            sleep 10                            # Wait for the server to be ready
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: false,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Playwright Local',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }
        stage('Deploy staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                npm ci
                npm install netlify-cli node-jq --no-save
                node_modules/.bin/netlify --version
                echo "Deploying to Netlify (staging)... Site ID: $NETLIFY_SITE_ID"
                node_modules/.bin/netlify status
                node_modules/.bin/netlify deploy --dir=build  --message="Deploy from Jenkins" --no-build --json > deploy-output.json
                '''
                script {
                    env.CI_ENVIRONMENT_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                }
            }
        }
        stage(' staging E2E Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.CI_ENVIRONMENT_URL}"
            }
            steps {
                sh '''
                    npm ci
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright E2E',
                useWrapperFileDirectly: true
            ])
                }
            }
        }
        stage('Approval') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    input message: 'Do you approve this deployment?', ok: 'Yes, I am sure!'
                }
            }
        }
        stage('Deploy prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                npm install netlify-cli --no-save
                node_modules/.bin/netlify --version
                echo "Deploying to Netlify... (production) Site ID: $NETLIFY_SITE_ID"
                node_modules/.bin/netlify status
                node_modules/.bin/netlify deploy --dir=build --prod --message="Deploy from Jenkins" --no-build
                '''
            }
        }
        stage(' prod E2E Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://bespoke-sable-fe6e32.netlify.app'
            }
            steps {
                sh '''
                    npm ci
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright E2E',
                useWrapperFileDirectly: true
            ])
                }
            }
        }
    }
}

// I've removed the invalid --skip-existing-files flag while keeping the essential flags:

// --dir=build: Specifies the directory to deploy
// --prod: Deploys to production
// --message="Deploy from Jenkins": Adds a deployment message
// --no-build: Prevents Netlify from trying to run another build
