pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'b06ff22f-332b-4260-b782-89691b0a075e'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
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
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
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
                    image 'node:18'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy-url' deploy-output.json", returnStdout: true)
                }
            }     
        }


        stage('Staging e2e') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test --reporter=html --output=test-results
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

        stage('Approval') {
            steps {
                timeout(15) {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'node:18'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to prod. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod e2e') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://preeminent-tiramisu-7d3109.netlify.app'
            }

            steps {
                sh '''
                    npx playwright test --reporter=html --output=test-results
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
