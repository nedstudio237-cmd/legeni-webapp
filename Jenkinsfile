// Projet CI/CD Legeni WebApp – Lot 4 (intégration continue) + Lot 5 (déploiement continu)
//   toutes branches / PR : qualité en parallèle -> build -> scan Trivy           (aucun déploiement)
//   main uniquement       : + push Docker Hub -> staging (Ansible) -> validation -> prod (Ansible) -> tag Git
// Aucun secret ici : uniquement des credentials Jenkins (registry-credentials, kubeconfig, ansible-vault-pass, github).
pipeline {
    agent any

    environment {
        IMAGE = 'nedstudio/legeni-webapp'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()                 // jamais deux déploiements du même job en même temps
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Préparation') {
            steps {
                script {
                    // Tag traçable <VERSION>-<numéro de build>, jamais « latest »
                    env.APP_VERSION = "${readFile('VERSION').trim()}-${env.BUILD_NUMBER}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} – ${env.APP_VERSION}"
                }
            }
        }

        stage('Qualité') {
            parallel {
                stage('Tests + couverture') {
                    agent { docker { image 'python:3.12-slim'; args '-e HOME=/tmp'; reuseNode true } }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            /tmp/venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('flake8') {
                    agent { docker { image 'python:3.12-slim'; args '-e HOME=/tmp'; reuseNode true } }
                    steps {
                        sh '''
                            python -m venv /tmp/venv-lint
                            /tmp/venv-lint/bin/pip install -q flake8==7.1.1
                            mkdir -p reports
                            /tmp/venv-lint/bin/flake8 --output-file=reports/flake8.txt --tee .
                        '''
                    }
                    post { always { recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')]) } }
                }
                stage('hadolint') {
                    steps {
                        sh '''
                            mkdir -p reports
                            docker run --rm -i hadolint/hadolint:v2.12.0 hadolint -f json - < Dockerfile > reports/hadolint.json
                        '''
                    }
                    post { always { recordIssues(tools: [hadoLint(pattern: 'reports/hadolint.json')]) } }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        withCredentials([file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
                            dir('ansible') {
                                sh '''
                                    ansible-playbook provision.yml --syntax-check
                                    ansible-playbook deploy.yml --syntax-check --vault-password-file "$VAULT_PASS"
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$APP_VERSION -t $IMAGE:$APP_VERSION .'
            }
        }

        stage('Scan Trivy') {
            steps {
                // quality gate : échec si une vulnérabilité CRITICAL corrigeable est trouvée
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
                        aquasec/trivy:0.57.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$APP_VERSION
                '''
            }
        }

        stage('Push Docker Hub') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                        docker push $IMAGE:$APP_VERSION
                        docker logout
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps {
                deployTo('staging')
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Déployer ${env.APP_VERSION} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps {
                deployTo('prod')
            }
        }

        stage('Tag Git') {
            when { branch 'main' }
            steps {
                // un tag v<VERSION> par mise en production (créé une seule fois par VERSION)
                withCredentials([usernamePassword(credentialsId: 'github',
                                                  usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        TAG=v$(cat VERSION)
                        REMOTE=https://$GH_USER:$GH_TOKEN@github.com/nedstudio237-cmd/legeni-webapp.git
                        if git ls-remote --tags "$REMOTE" | grep -q "refs/tags/$TAG$"; then
                            echo "Tag $TAG déjà présent"
                        else
                            git -c user.name=jenkins -c user.email=jenkins@legeni.local \
                                tag -a "$TAG" -m "Mise en production $APP_VERSION"
                            git push "$REMOTE" "$TAG"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.JOB_NAME} ${env.APP_VERSION} OK"
        }
        failure {
            emailext to: 'nedstudio237@gmail.com',
                     subject: "❌ Échec ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: "Le pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.APP_VERSION}) a échoué.\nConsole : ${env.BUILD_URL}console",
                     attachLog: true
        }
        cleanup {
            cleanWs()
        }
    }
}

// Déploiement Kubernetes par Ansible (Lot 3) : rolling update, smoke test, rollback automatique
def deployTo(String targetEnv) {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                     file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
        dir('ansible') {
            sh """
                ansible-playbook deploy.yml -e target_env=${targetEnv} -e app_version=\$APP_VERSION \
                    --vault-password-file "\$VAULT_PASS"
            """
        }
    }
}
