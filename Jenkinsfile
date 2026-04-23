pipeline {
    agent any

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
                    npm  test
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
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test
                '''
            }
        }
    }

    post{
        always{
           junit 'test-results/junit.xml'
        }
    }
}