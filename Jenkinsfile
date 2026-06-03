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
                sh  '''
                    test -f build/index.html
                    npm install jest-junit --save-dev
                    npm test
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
                sh  '''
                    npm install serve
                    npx playwright install
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test --reporter=html
                '''
            }
        }
    }

    post{
        always{
            junit allowEmptyResults: true, testResults: 'test-results/junit.xml'
        }
    }
}