pipeline {
    agent any

    stages {

        stage('Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/sergioglezz/CasoPractico1-4.git'
                echo WORKSPACE
                
                dir('config-temp') {
                    git branch: 'production', url: 'https://github.com/sergioglezz/todo-list-aws-config.git'
                }
                
                sh '''
                    cp config-temp/samconfig.toml .
                    rm -rf config-temp
                '''
                
                sh 'ls -la'
                stash name: 'source-code', includes: '**/*'
            }
        }

        stage('Deploy') {
            steps {
                sh 'sam build'
                
                sh '''
                    sam deploy \
                        --config-file samconfig.toml \
                        --config-env production \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset
                '''
                
                script {
                    env.API_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()
                    echo "API URL: ${env.API_URL}"
                }
            }
        }

        stage('Rest Test') {
            steps {
                sh 'pip3 install pytest requests --break-system-packages -q'
                sh """
                    export PATH=\$PATH:/var/lib/jenkins/.local/bin
                    export BASE_URL='${env.API_URL}'
                    pytest test/integration/todoApiTest.py -m readonly --junitxml=result-rest.xml -v
                """
                junit 'result-rest.xml'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline CD completado con exito!'
        }
        failure {
            echo 'Pipeline CD fallido!'
        }
    }
}