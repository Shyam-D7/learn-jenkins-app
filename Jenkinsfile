pipeline {
    agent any

    environment {
        BUILD_FILE_NAME = 'index.html'
        NETLIFY_SITE_ID = '4115a522-c293-46db-8d47-15d8987b0178'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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
                            # test -f build/$BUILD_FILE_NAME
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
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                            // args '-u root:root'
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
                            publishHTML (
                                [
                                    allowMissing: false, 
                                    alwaysLinkToLastBuild: false, 
                                    keepAll: false, 
                                    reportDir: 'playwright-report', 
                                    reportFiles: 'index.html', 
                                    reportName: 'Playwright Local Report', 
                                    reportTitles: '', 
                                    useWrapperFileDirectly: true
                                ]
                            )
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
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: 'node_modules/.bin/node-jq -r ".deploy_url" deploy-output.json', returnStdout: true)
                }
            }
            
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    // args '-u root:root'
                }
            }
            environment {
                // CI_ENVIRONMENT_URL = "$env.STAGING_URL"
                CI_ENVIRONMENT_URL = ${env.STAGING_URL}
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML (
                        [
                            allowMissing: false, 
                            alwaysLinkToLastBuild: false, 
                            keepAll: false, 
                            reportDir: 'playwright-report', 
                            reportFiles: 'index.html', 
                            reportName: 'Staging E2E Report', 
                            reportTitles: '', 
                            useWrapperFileDirectly: true
                        ]
                    )
                }
            }
        } 

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'SECONDS') {
                    input message: 'Ready to deploy?', ok: 'Yes, deploy ...'
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
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    // args '-u root:root'
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://cool-starlight-5f85e9.netlify.app'
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML (
                        [
                            allowMissing: false, 
                            alwaysLinkToLastBuild: false, 
                            keepAll: false, 
                            reportDir: 'playwright-report', 
                            reportFiles: 'index.html', 
                            reportName: 'Prod E2E Report', 
                            reportTitles: '', 
                            useWrapperFileDirectly: true
                        ]
                    )
                }
            }
        }        

        
    }

    post {
        always {
            junit 'jest-results/junit.xml'
            publishHTML (
                [
                    allowMissing: false, 
                    alwaysLinkToLastBuild: false, 
                    keepAll: false, 
                    reportDir: 'playwright-report', 
                    reportFiles: 'index.html', 
                    reportName: 'Playwright HTML Report', 
                    reportTitles: '', 
                    useWrapperFileDirectly: true
                ]
            )
        }
    }
}
