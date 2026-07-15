pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '5c48bf24-88bc-4964-a34b-d6ba5dd7c1a9'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = '1.2.3'
    }

    stages {
        stage('Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            test -f build/index.html
                            npm install jest-junit --save-dev
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'test-results/junit.xml'
                        }
                    }
                }

                stage('E2E Local') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                            reuseNode true
                        }
                    }
                    // REMOVIDO: environment vazio
                    steps {
                        script {
                            sh '''
                                echo "=== Starting E2E Local Tests ==="

                                npm install serve
                                npx playwright install

                                if [ ! -d "build" ]; then
                                    echo "❌ Build directory not found!"
                                    exit 1
                                fi

                                echo "Starting server on port 3000..."
                                node_modules/.bin/serve -s build -l 3000 > server.log 2>&1 &
                                SERVER_PID=$!
                                echo "Server PID: $SERVER_PID"

                                echo "Waiting for server to be ready..."
                                MAX_RETRIES=15
                                RETRY_COUNT=0
                                SERVER_READY=false

                                while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                                    if curl -s -f http://localhost:3000 > /dev/null 2>&1; then
                                        echo "✅ Server is ready on port 3000!"
                                        SERVER_READY=true
                                        break
                                    fi
                                    RETRY_COUNT=$((RETRY_COUNT+1))
                                    echo "Attempt $RETRY_COUNT/$MAX_RETRIES: Server not ready yet..."
                                    sleep 2
                                done

                                if [ "$SERVER_READY" = false ]; then
                                    echo "❌ Server failed to start after $MAX_RETRIES attempts"
                                    cat server.log || echo "No logs available"
                                    kill $SERVER_PID 2>/dev/null || true
                                    exit 1
                                fi

                                echo "Running Playwright tests..."
                                npx playwright test --reporter=html --output=playwright-local
                                TEST_EXIT_CODE=$?

                                echo "Stopping server (PID: $SERVER_PID)..."
                                kill $SERVER_PID 2>/dev/null || true
                                pkill -f "serve -s build" || true

                                exit $TEST_EXIT_CODE
                            '''
                        }
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: true,
                                alwaysLinkToLastBuild: false,
                                icon: '',
                                keepAll: false,
                                reportDir: 'playwright-local',
                                reportFiles: 'index.html',
                                reportName: 'E2E Local Tests',
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh '''
                        echo "Current directory: $(pwd)"
                        echo "Workspace: $WORKSPACE"
                        ls -la

                        cd $WORKSPACE
                        ls -la

                        npm init -y
                        npm install netlify-cli@20.1.1
                        node_modules/.bin/netlify --version
                        echo "Deploying to staging. Site Id: $NETLIFY_SITE_ID"

                        node_modules/.bin/netlify status
                    '''

                    def deployUrl = sh(
                        script: '''
                            node_modules/.bin/netlify deploy --dir=build --json > /tmp/deploy-output.json

                            DEPLOY_URL=$(grep -o '"deploy_url":"[^"]*"' /tmp/deploy-output.json | cut -d'"' -f4)

                            if [ -z "$DEPLOY_URL" ]; then
                                DEPLOY_URL=$(grep -o 'https://[^"]*\\.netlify\\.app' /tmp/deploy-output.json | head -1)
                            fi

                            echo "$DEPLOY_URL"
                            rm -f /tmp/deploy-output.json
                        ''',
                        returnStdout: true
                    ).trim()

                    env.STAGING_URL = deployUrl

                    env.DEPLOY_ID = sh(
                        script: '''
                            node_modules/.bin/netlify deploy --dir=build --json | \
                            grep -o '"deploy_id":"[^"]*"' | \
                            cut -d'"' -f4
                        ''',
                        returnStdout: true
                    ).trim()

                    if (!env.STAGING_URL) {
                        error "❌ STAGING_URL está vazio! Deploy falhou."
                    }

                    echo "✅ Deploy ID: ${env.DEPLOY_ID}"
                    echo "✅ Staging URL: ${env.STAGING_URL}"
                }
            }
        }

        stage('E2E Staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }
            steps {
                script {
                    sh '''
                        echo "=== Testing Staging Environment ==="
                        echo "CI_ENVIRONMENT_URL: ${CI_ENVIRONMENT_URL}"

                        echo "Checking if staging is accessible..."
                        MAX_RETRIES=10
                        RETRY_COUNT=0
                        while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                            if curl -s -f ${CI_ENVIRONMENT_URL} > /dev/null 2>&1; then
                                echo "✅ Staging is accessible!"
                                break
                            fi
                            RETRY_COUNT=$((RETRY_COUNT+1))
                            echo "Attempt $RETRY_COUNT/$MAX_RETRIES: Staging not ready yet..."
                            sleep 5
                        done

                        npm install @playwright/test@latest --save-dev
                        npx playwright install
                        npx playwright test --reporter=html --output=playwright-staging
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        icon: '',
                        keepAll: false,
                        reportDir: 'playwright-staging',
                        reportFiles: 'index.html',
                        reportName: 'E2E Staging Tests',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: 'Ready to deploy to production?'
                }
            }
        }

        stage('Deploy Production') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Workspace: $WORKSPACE"
                    ls -la

                    cd $WORKSPACE
                    ls -la

                    npm init -y
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site Id: $NETLIFY_SITE_ID"

                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('E2E Production') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://genuine-malasada-895bd3.netlify.app'
            }
            steps {
                script {
                    sh '''
                        echo "=== Testing Production Environment ==="
                        echo "CI_ENVIRONMENT_URL: ${CI_ENVIRONMENT_URL}"

                        echo "Checking if production is accessible..."
                        MAX_RETRIES=10
                        RETRY_COUNT=0
                        while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                            if curl -s -f ${CI_ENVIRONMENT_URL} > /dev/null 2>&1; then
                                echo "✅ Production is accessible!"
                                break
                            fi
                            RETRY_COUNT=$((RETRY_COUNT+1))
                            echo "Attempt $RETRY_COUNT/$MAX_RETRIES: Production not ready yet..."
                            sleep 5
                        done

                        npm install @playwright/test@latest --save-dev
                        npx playwright install
                        npx playwright test --reporter=html --output=playwright-production
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        icon: '',
                        keepAll: false,
                        reportDir: 'playwright-production',
                        reportFiles: 'index.html',
                        reportName: 'E2E Production Tests',
                        reportTitles: '',
                        useWrapperFileDirectly: true
                    ])
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed!"
        }
        success {
            echo "🎉 Pipeline succeeded! All tests passed!"
        }
        failure {
            echo "❌ Pipeline failed! Check the logs above."
        }
    }
}