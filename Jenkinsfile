pipeline {
    agent any

    options { skipDefaultCheckout() }

    stages {
        stage('SetUp'){
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install flake8 bandit pytest boto3 requests
                '''
            }
        }
        stage( 'Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/migueldonaire/todo-list-aws.git'
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    flake8 src/ --format=html --htmldir=flake8-report || true
                    bandit -r src/ -f html -o bandit-report.html || true
                '''
            }
            post {
                always {
                    publishHTML([reportDir: '.', reportFiles: 'flake8-report/index.html', reportName: 'Flake8'])
                    publishHTML([reportDir: '.', reportFiles: 'bandit-report.html', reportName: 'Bandit'])
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    sam build
                    sam deploy --config-env staging --no-fail-on-empty-changeset --no-progressbar
                '''
            }
        }

        stage('Rest Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    BASE_URL=$(aws cloudformation describe-stacks \
                        --stack-name todo-list-aws-staging \
                        --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                        --region us-east-1 \
                        --output text)
                    export BASE_URL
                    pytest test/integration/todoApiTest.py -v
                '''
            }
        }
        
        stage('Promote') {
            steps {
                sh '''
                    git checkout master
                    git merge develop
                    git push origin master
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}