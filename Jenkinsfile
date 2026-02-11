pipeline {
    agent any

    options { skipDefaultCheckout() }

    stages {
        stage('Get Code') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/migueldonaire/todo-list-aws.git',
                    credentialsId: 'github-credentials'
                stash name:'code', includes:'**'
                script {
                    deleteDir()
                }
            }
        }

        stage('Static Test') {
            steps {
                unstash name:'code'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install flake8 flake8-html bandit
                    flake8 src/ --format=html --htmldir=flake8-report || true
                    bandit -r src/ -f html -o bandit-report.html || true
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'flake8-report',
                        reportFiles: 'index.html',
                        reportName: 'Flake8'
                    ])
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'bandit-report.html',
                        reportName: 'Bandit'
                    ])
                    deleteDir()
                }
            }
        }

        stage('Deploy') {
            steps {
                unstash name:'code'
                sh '''
                    sam build
                    sam deploy --config-env staging --no-fail-on-empty-changeset --no-progressbar
                '''
            }
        }

        stage('Rest Test') {
            steps {
                unstash name:'code'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install pytest boto3 requests
                    BASE_URL=$(aws cloudformation describe-stacks \
                        --stack-name todo-list-aws-staging \
                        --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                        --region us-east-1 \
                        --output text)
                    export BASE_URL
                    pytest test/integration/todoApiTest.py -v
                '''
                script {
                    deleteDir()
                }
            }
        }

        stage('Promote') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/migueldonaire/todo-list-aws.git',
                    credentialsId: 'github-credentials'
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/migueldonaire/todo-list-aws.git

                        # Obtener última versión del CHANGELOG
                        LAST_VERSION=$(grep -oP '(?<=## \\[)[0-9]+\\.[0-9]+\\.[0-9]+' CHANGELOG.md | head -1)
                        echo "Última versión: $LAST_VERSION"

                        # Incrementar patch version
                        MAJOR=$(echo $LAST_VERSION | cut -d. -f1)
                        MINOR=$(echo $LAST_VERSION | cut -d. -f2)
                        PATCH=$(echo $LAST_VERSION | cut -d. -f3)
                        NEW_PATCH=$((PATCH + 1))
                        NEW_VERSION="${MAJOR}.${MINOR}.${NEW_PATCH}"
                        echo "Nueva versión: $NEW_VERSION"

                        # Fecha actual
                        TODAY=$(date +%Y-%m-%d)

                        # Crear nueva entrada en CHANGELOG
                        sed -i "/^## \\[${LAST_VERSION}\\]/i ## [${NEW_VERSION}] - ${TODAY}\\n### Changed\\n- Actualizado automáticamente por Jenkins (Build #${BUILD_NUMBER})\\n" CHANGELOG.md

                        # Commit en develop
                        git add CHANGELOG.md
                        git commit -m "chore: bump version to ${NEW_VERSION} [Jenkins]"
                        git push origin develop

                        # Merge a master
                        git checkout master
                        git merge develop
                        git push origin master
                    '''
                }
                script {
                    deleteDir()
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
