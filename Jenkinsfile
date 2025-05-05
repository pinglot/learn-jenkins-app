pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd963ac05-ebd8-4ecf-a01a-c64378f700ee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {
        stage('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
            }
        }

        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            steps {
                sh '''
                    aws --version
                    // aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                    // aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                    // aws configure set default.region us-east-1
                    // aws s3 cp s3://my-bucket-name/ . --recursive
                '''
            }
        }

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
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            serve -s build &
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
                    image 'my-playwright'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    npm --version
                    echo "Deploying to staging"
                    netlify --version
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    def deployUrl = sh(
                        script: 'node-jq -r .deploy_url deploy-output.json', // -r for raw output
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
                    image 'my-playwright'
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
                    netlify --version
                    netlify status
                    netlify deploy --dir=build --prod 
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
