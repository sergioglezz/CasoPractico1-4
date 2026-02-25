pipeline {
    agent any

    stages {

        stage('Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/sergioglezz/CasoPractico1-4.git'
                echo WORKSPACE
                
                dir('config-temp') {
                    git branch: 'staging', url: 'https://github.com/sergioglezz/todo-list-aws-config.git'
                }
                
                sh '''
                    cp config-temp/samconfig.toml .
                    rm -rf config-temp
                '''
                
                sh 'ls -la'
                stash name: 'source-code', includes: '**/*'
            }
        }

        stage('Static Test') {
            steps {
                sh 'pip3 install flake8 bandit --break-system-packages -q'

                sh '/var/lib/jenkins/.local/bin/flake8 --exit-zero --format=pylint src > flake8.out'
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                sh '/var/lib/jenkins/.local/bin/bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }
        
        stage('Deploy') {
            steps {
                sh 'sam build'
                
                sh '''
                    sam deploy \
                        --config-file samconfig.toml \
                        --config-env staging \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset
                '''
                
                script {
                    env.API_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
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
                    pytest test/integration/todoApiTest.py --junitxml=result-rest.xml -v
                """
                junit 'result-rest.xml'
            }
        }

        stage('Promote') {
            agent any
            steps {
                unstash 'source-code'
                unstash 'test-results'
                junit 'result-rest.xml'
                
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
                        git fetch origin
                        
                        # Intentar merge
                        if git merge develop --no-edit; then
                            echo "Merge exitoso sin conflictos"
                        else
                            echo "Conflicto detectado, resolviendo..."
                            # Mantener ambos Jenkinsfiles de master
                            git checkout --ours Jenkinsfile
                            git checkout --ours Jenkinsfile_agentes
                            git add Jenkinsfile Jenkinsfile_agentes
                            git commit -m "Merge develop manteniendo Jenkinsfiles de master"
                        fi
                        
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