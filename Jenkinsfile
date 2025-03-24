pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd963ac05-ebd8-4ecf-a01a-c64378f700ee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
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
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
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
                            test -e build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }                    
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.51.0-noble'
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
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {

            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.51.0-noble'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    npm --version
                    npm install netlify-cli node-jq
                    echo "Deploying to staging"
                    node_modules/.bin/netlify --version
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    def deployUrl = sh(
                        script: 'node_modules/.bin/node-jq -r .deploy_url deploy-output.json', // -r for raw output
                        returnStdout: true
                    ).trim()
        
                    env.CI_ENVIRONMENT_URL = deployUrl
                }                
                sh '''
                    echo CI environment URL: $CI_ENVIRONMENT_URL
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright staging smoke test Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy prod') {

            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.51.0-noble'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://soft-pegasus-585f6d.netlify.app/'
            }

            steps {
                sh '''
                    npm --version
                    npm install netlify-cli
                    echo "Deploying to prod"
                    node_modules/.bin/netlify --version
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod 
                    echo CI environment URL: $CI_ENVIRONMENT_URL
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright PROD smoke test Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

    }
}
