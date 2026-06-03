pipeline {
    agent any

    stages {
        stage('Test'){
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    test -f build/index.html

                    # Instala dependências de teste
                    npm install --save-dev jest-junit

                    # Configura variável para o reporter
                    export JEST_JUNIT_OUTPUT_DIR=test-results
                    export JEST_JUNIT_OUTPUT_NAME=junit.xml

                    # Roda testes e gera relatório
                    CI=true npm test -- --testResultsProcessor="jest-junit" --ci

                    # Verifica se o arquivo foi criado
                    if [ -f test-results/junit.xml ]; then
                        echo "✅ Test report generated successfully"
                        cat test-results/junit.xml | head -n 5
                    else
                        echo "⚠️ No test report generated"
                        # Cria um relatório vazio para não falhar o pipeline
                        mkdir -p test-results
                        echo '<?xml version="1.0" encoding="UTF-8"?><testsuites></testsuites>' > test-results/junit.xml
                    fi
                '''
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
                sh '''
                    npm install serve
                    npx playwright install --with-deps

                    node_modules/.bin/serve -s build &
                    sleep 10

                    # Cria diretório para relatórios
                    mkdir -p test-results/e2e

                    # Roda testes com múltiplos reporters
                    npx playwright test --reporter=html,junit --output=test-results/e2e/

                    # Move o relatório JUnit para o local esperado
                    if [ -f test-results/e2e/junit.xml ]; then
                        cp test-results/e2e/junit.xml test-results/junit-e2e.xml
                        echo "✅ E2E test report generated"
                    fi
                '''
            }
        }
    }

    post{
        always{
            script {
                // Publica todos os relatórios JUnit encontrados
                junit allowEmptyResults: true,
                      testResults: '**/test-results/**/junit*.xml, **/junit.xml'

                // Publica artefatos HTML do Playwright (opcional)
                archiveArtifacts allowEmptyArchive: true,
                                artifacts: '**/playwright-report/**/*, **/test-results/**/*.html'
            }
        }
    }
}