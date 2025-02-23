pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    environment {
        AWS_REGION = 'us-east-1'
        DEPLOY_ENV = (env.BRANCH_NAME == 'main') ? 'production' : 'staging'
    }

    stages {
        stage('GetCode') {
            // agent { label 'main-agent' }
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/mbedia94/unir-todo-list-aws',
                    ]]
                ])
                sh '''
                    whoami
                    hostname
                    echo ${WORKSPACE}
                '''
            }
        }

        stage('Static Test') {
            // agent { label 'test-agent' }
            when {
                branch 'develop'
            }
            steps {
                // checkout scm
                sh '''
                    whoami
                    hostname
                    echo ${WORKSPACE}
                    
                    # Crear entorno virtual y activar
                    python3 -m venv unir
                    . unir/bin/activate
                    export PATH="${WORKSPACE}/unir/bin:$PATH"
                    pip install --upgrade pip
                    pip install -r requirements.txt

                    # Ejecutar Flake8 en /src
                    flake8 --exit-zero --format=pylint src > flake8.out
                '''
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                sh '''
                    # Ejecutar Bandit en /src
                    . unir/bin/activate
                    export PATH="${WORKSPACE}/unir/bin:$PATH"
                    bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                '''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }

        stage('Deploy') {
            // agent { label 'deploy-agent' }
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                // checkout scm
                sh '''
                    whoami
                    hostname
                    echo ${WORKSPACE}

                    # Configurar AWS CLI si es necesario
                    aws configure set region ${AWS_REGION}

                    # Construcción del proyecto
                    sam build

                    # Validar la plantilla de CloudFormation
                    sam validate

                    # Desplegar usando la configuración del archivo samconfig.toml
                    sam deploy --config-env ${DEPLOY_ENV} --no-confirm-changeset --force-upload --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test') {
            // agent { label 'test-agent' }
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                // checkout scm
                sh '''
                    whoami
                    hostname
                    echo ${WORKSPACE}

                    # Crear entorno virtual y activar
                    python3 -m venv unir
                    . unir/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt

                     BASE_URL=$(aws cloudformation describe-stacks \
                        --stack-name todo-list-aws-${DEPLOY_ENV} \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$BASE_URL" ]; then
                        echo "ERROR: No se pudo obtener la BASE_URL de la API."
                        exit 1
                    fi

                    echo "BASE_URL obtenida: $BASE_URL"
                    export BASE_URL

                    if [ "${DEPLOY_ENV}" == "production" ]; then
                        pytest --junitxml=result-integration.xml -m "read_only" test/integration/todoApiTest.py
                    else
                        pytest --junitxml=result-integration.xml test/integration/todoApiTest.py
                    fi
                '''
                junit 'result-integration.xml'
            }
        }

        stage('Promote') {
            // agent { label 'main-agent' }
            when {
                branch 'develop'
            }
            environment {
                GIT_TOKEN = credentials('GIT_TOKEN')
            }
            steps {
                script {
                    try {
                        sh '''
                            git config --global user.email "jenkins@unir.com"
                            git config --global user.name "Jenkins CI"

                            git fetch origin

                            if ! git rev-parse --verify develop; then
                                git checkout -b develop origin/develop
                            else
                                git checkout develop
                            fi

                            git checkout main

                            git merge --no-ff origin/develop -m "Merge de develop a main"

                            git push https://${GIT_TOKEN}@github.com/mbedia94/unir-todo-list-aws.git main
                        '''
                    } catch (Exception e) {
                        error("Fallo en la promoción de la versión a producción.")
                    }
                }
            }
        }


    }
}
