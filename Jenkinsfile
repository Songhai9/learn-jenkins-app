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
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                            publishHTML(
                                allowMissing: false, 
                                alwaysLinkToLastBuild: true, 
                                icon: '', 
                                keepAll: true, 
                                reportDir: 'playwright-report', 
                                reportFiles: 'index.html', 
                                reportName: 'playwright HTML Local Report', 
                                reportTitles: '', 
                                useWrapperFileDirectly: false
                            )
                        }
                    }
                }
            }
        }

        stage('Deploy') {
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
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
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
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    junit 'jest-results/junit.xml'
                    publishHTML(
                        allowMissing: false, 
                        alwaysLinkToLastBuild: true, 
                        icon: '', 
                        keepAll: true, 
                        reportDir: 'playwright-report', 
                        reportFiles: 'index.html', 
                        reportName: 'playwright HTML e2e Report', 
                        reportTitles: '', 
                        useWrapperFileDirectly: false
                    )
                }
            }
        }
    
    }
}
