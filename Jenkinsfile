pipeline {
    agent { label 'principal' }

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    environment {
        AWS_REGION = 'us-east-1'
        DEPLOY_ENV = 'staging'
    }

    stages {
        stage('GetCode') {
            // agent { label 'main-agent' }
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/develop']],
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
                    
                    # Ejecutar Flake8 en /src
                    flake8 --exit-zero --format=pylint src > flake8.out
                '''
                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                sh '''
                    # Ejecutar Bandit en /src
                    bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                '''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
            }
        }

        stage('Deploy') {
            // agent { label 'deploy-agent' }
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

                    # Ejecutar pruebas de integración con pytest
                    PYTHONPATH=$(pwd) pytest --junitxml=result-integration.xml test/integration/todoApiTest.py
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
                GIT_REPO = "https://${GIT_TOKEN}@github.com/mbedia94/unir-todo-list-aws.git"
            }
            steps {
                script {
                    try {
                        sh '''
                            git config --global user.email "jenkins@unir.com"
                            git config --global user.name "Jenkins CI"

                            git fetch origin

                            git checkout main

                            git merge --no-ff develop -m "Merge de develop a main"

                            git push ${GIT_REPO} main
                        '''
                    } catch (Exception e) {
                        error("Fallo en la promoción de la versión a producción.")
                    }
                }
            }
        }


    }
}
