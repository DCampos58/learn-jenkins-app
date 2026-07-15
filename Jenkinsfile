pipeline {
    agent any

    environment{
        NETLIFY_SITE_ID = '5c48bf24-88bc-4964-a34b-d6ba5dd7c1a9'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION= '1.2.3'
    }

    stages {
        // this is a comment
        /*
        line 1
        line 2
        posso comentar stages
        dentro de sh posso comentar linhas com #
        */

        //comentado já que no jenkins tenho o build do projeto e o build demora muito a ser feito
        /*
        stage('Build') {
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo 'Small change'
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        */


        stage('Tests'){
            parallel{
                    stage('Unit Test'){
                        agent{
                            docker{
                                image 'node:18-alpine'
                                reuseNode true
                            }
                        }

                        steps{
                            sh  '''
                                test -f build/index.html
                                npm install jest-junit --save-dev
                                npm test
                            '''
                        }
                        post{
                           always{
                                junit allowEmptyResults: true, testResults: 'test-results/junit.xml'
                           }
                        }
                    }
                stage('E2E'){
                    agent{
                        docker{
                            image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                            reuseNode true
                        }
                    }
                    steps{
                        sh  '''
                            npm install serve
                            npx playwright install

                            # Iniciar servidor em background (apenas UM servidor)
                            node_modules/.bin/serve -s build -l 3000 &

                            # Aguardar servidor ficar disponível
                            echo "Waiting for server to be ready..."
                            for i in 1 2 3 4 5 6 7 8 9 10; do
                                if curl -s http://localhost:3000 > /dev/null; then
                                    echo "Server is ready!"
                                    break
                                fi
                                echo "Attempt $i: Server not ready yet..."
                                sleep 2
                            done

                            # Executar testes (sem o segundo serve)
                            npx playwright test --reporter=html --output=playwright-local
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: true,  // Mudar para true
                                        alwaysLinkToLastBuild: false,
                                        icon: '',
                                        keepAll: false,
                                        reportDir: 'playwright-local',
                                        reportFiles: 'index.html',
                                        reportName: 'playwright HTML Local',
                                        reportTitles: '',
                                        useWrapperFileDirectly: true])
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

                script {
                    // Executa o deploy e extrai a URL usando apenas shell
                    def deployUrl = sh(
                        script: '''
                            node_modules/.bin/netlify deploy --dir=build --json > /tmp/deploy-output.json

                            # Extrair URL usando grep e cut (nativos do Alpine)
                            DEPLOY_URL=$(grep -o '"deploy_url":"[^"]*"' /tmp/deploy-output.json | cut -d'"' -f4)

                            # Se falhar, tentar método alternativo
                            if [ -z "$DEPLOY_URL" ]; then
                                DEPLOY_URL=$(grep -o 'https://[^"]*\\.netlify\\.app' /tmp/deploy-output.json | head -1)
                            fi

                            echo "$DEPLOY_URL"

                            # Limpar arquivo temporário
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


        stage('Stage E2E'){
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                    reuseNode true
                }
            }

            environment{
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps{
                sh  '''
                    # Instalar/atualizar Playwright para a versão compatível
                    npm install @playwright/test@latest --save-dev
                    npx playwright install
                    npx playwright test --reporter=html --output=playwright-report
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
        /*
        stage('Approval') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    input 'Ready to deploy?'
                }
            }
        }
        */
        stage('Deploy prod'){
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.58.2-noble'
                    reuseNode true
                }
            }

            environment{
                CI_ENVIRONMENT_URL = 'https://genuine-malasada-895bd3.netlify.app'
            }

            steps{
                sh  '''
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


                    # Instalar/atualizar Playwright para a versão compatível
                    npm install @playwright/test@latest --save-dev
                    npx playwright install
                    npx playwright test --reporter=html --output=playwright-report
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'playwright HTML E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

    }
}