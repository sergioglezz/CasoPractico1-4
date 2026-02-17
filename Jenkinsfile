pipeline {
    agent any

    stages {

        stage('Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/sergioglezz/CasoPractico1-4.git'
                echo WORKSPACE
                sh 'ls -la'
            }
        }

        stage('Static Test') {
            steps {
                sh 'pip3 install flake8 bandit --break-system-packages -q'

                // Flake8 - análisis de estilo
                sh '/var/lib/jenkins/.local/bin/flake8 --exit-zero --format=pylint src > flake8.out'
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                // Bandit - análisis de seguridad
                sh '/var/lib/jenkins/.local/bin/bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }

        stage('Deploy') {
            steps {
                sh 'sam build'
                sh '''
                   sam deploy \
                    --stack-name todo-list-aws \
                    --parameter-overrides Stage=staging \
                    --no-confirm-changeset \
                    --no-fail-on-empty-changeset \
                    --resolve-s3 \
                    --capabilities CAPABILITY_IAM \
                    --region us-east-1
                '''
                script {
                    env.API_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text --region us-east-1",
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
                    export BASE_URL='${env.API_URL}'
                    pytest test/integration/todoApiTest.py --junitxml=result-rest.xml -v
                """
                junit 'result-rest.xml'
            }
        }

        stage('Promote') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        git config user.email "jenkins@devops.com"
                        git config user.name "Jenkins CI"
                        git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/sergioglezz/CasoPractico1-3.git
                        git checkout master
                        git merge develop --no-edit
                        git push origin master
                    '''
                }
            }
        }

    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline CI completado con exito!'
        }
        failure {
            echo 'Pipeline CI fallido!'
        }
    }
}