pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'b06ff22f-332b-4260-b782-89691b0a075e'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        stage('Docker') {
            sh 'docker build -t my-playwright .'
        }
        
        stage('build') {
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

        stage('Run tests') {
            parallel {
                stage('Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        echo "Test stage"
                        sh '''
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('e2e') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html --output=test-results
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                            publishHTML(target: [
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Playwright Local Report',
                                keepAll: true,
                                alwaysLinkToLastBuild: true
                            ])
                        }
                    }
                }
            }
        }


        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL='STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    npm install netlify-cli node-jq
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
                    npx playwright test --reporter=html 
                '''
            }
            post {
                always {
                    junit 'jest-results/junit.xml'
                    publishHTML(target: [
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Playwright E2E Report',
                        keepAll: true,
                        alwaysLinkToLastBuild: true
                    ])
                }
            }
        }


        

        stage('Deploy prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://preeminent-tiramisu-7d3109.netlify.app'
            }

            steps {
                sh '''
                    npm install netlify-cli
                    netlify --version
                    echo "Deploying to prod. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    sleep 10
                    npx playwright test --reporter=html 
                '''
            }
            post {
                always {
                    junit 'jest-results/junit.xml'
                    publishHTML(target: [
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Playwright E2E Report',
                        keepAll: true,
                        alwaysLinkToLastBuild: true
                    ])
                }
            }
        }
    
    }
}

