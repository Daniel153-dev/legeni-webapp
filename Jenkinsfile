// TP 3.8 : intégration continue + déploiement continu (staging automatique, prod après validation)
pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKERHUB_ID',
            defaultValue: 'darknithadiene',
            description: "Votre identifiant Docker Hub"
        )
        booleanParam(
            name: 'PUSH',
            defaultValue: true,
            description: "Pousser l'image sur Docker Hub"
        )
    }

    environment {
        IMAGE   = "${params.DOCKERHUB_ID}/legeni-webapp"
        COMPOSE = "docker compose -f deploy/docker-compose.deploy.yml"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Initialisation') {
            steps {
                script {
                    def rawBranch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'main'
                    env.CURRENT_BRANCH = rawBranch.replaceFirst('^origin/', '')
                    env.VERSION = (env.CURRENT_BRANCH == 'main') ? "1.0.${env.BUILD_NUMBER}" : "${env.CURRENT_BRANCH.replaceAll('[^a-zA-Z0-9.-]', '-')}-${env.BUILD_NUMBER}"
                    echo "Branche : ${env.CURRENT_BRANCH} | Version du build : ${env.VERSION}"
                }
            }
        }

        stage('Tests unitaires') {
            steps {
                sh 'docker pull python:3.12-slim || true'
                sh 'docker build --target test -t legeni-webapp:test .'
            }
        }

        stage('Build image') {
            steps {
                sh '''
                    docker build \
                        --build-arg APP_VERSION=$VERSION \
                        -t $IMAGE:$VERSION \
                        -t $IMAGE:latest .
                '''
            }
        }

        stage("Test d'acceptance") {
            steps {
                script {
                    env.TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
                    env.CI_NETWORK = "ci-net-${env.BUILD_NUMBER}"
                }
                sh '''
                    docker network create $CI_NETWORK

                    docker run -d \
                        --name $TEST_CONTAINER \
                        --network $CI_NETWORK \
                        $IMAGE:$VERSION

                    docker run --rm \
                        --network $CI_NETWORK \
                        -e TEST_CONTAINER=$TEST_CONTAINER \
                        -e APP_VERSION=$VERSION \
                        python:3.12-slim python3 -c '
import urllib.request
import time
import os

container = os.environ["TEST_CONTAINER"]

for i in range(15):
    try:
        res = urllib.request.urlopen(
            "http://" + container + ":5000/health",
            timeout=3
        )
        if res.status == 200:
            print("Health check OK")
            break
    except Exception:
        time.sleep(1)
else:
    raise SystemExit("Health check failed")

version = os.environ["APP_VERSION"]

html = urllib.request.urlopen(
    "http://" + container + ":5000/"
).read().decode("utf-8")

assert "Version " + version in html, "Version non trouvee dans la page"

print("Validation de la version reussie !")
'
                '''
            }
        }

        stage('Déploiement staging') {
            environment {
                REDIS_PASSWORD = credentials('redis-password-staging')
            }
            steps {
                sh '''
                    IMAGE_REF=$IMAGE:$VERSION APP_ENV=staging APP_COLOR='#bf8700' PORT=8081 \
                        $COMPOSE -p legeni-staging up -d --wait
                '''
                sh './deploy/smoke-test.sh legeni-staging $VERSION'
            }
        }

        stage('Validation production') {
            when { expression { return env.CURRENT_BRANCH == 'main' } }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Déployer la version ${VERSION} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { expression { return env.CURRENT_BRANCH == 'main' } }
            environment {
                REDIS_PASSWORD = credentials('redis-password-prod')
            }
            steps {
                script {
                    env.PREVIOUS_IMAGE = sh(
                        script: "docker inspect -f '{{.Config.Image}}' legeni-prod-web-1 2>/dev/null || echo none",
                        returnStdout: true
                    ).trim()
                    echo "Version en production avant déploiement : ${env.PREVIOUS_IMAGE}"
                    try {
                        sh '''
                            IMAGE_REF=$IMAGE:$VERSION APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                $COMPOSE -p legeni-prod up -d --wait
                            ./deploy/smoke-test.sh legeni-prod $VERSION
                        '''
                    } catch (err) {
                        if (env.PREVIOUS_IMAGE != 'none') {
                            echo "Smoke test KO : retour à ${env.PREVIOUS_IMAGE}"
                            sh '''
                                IMAGE_REF=$PREVIOUS_IMAGE APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                    $COMPOSE -p legeni-prod up -d --wait
                            '''
                        }
                        error "Déploiement production échoué, rollback effectué"
                    }
                }
            }
        }

        stage('Push Docker Hub') {
            when {
                expression {
                    return env.CURRENT_BRANCH == 'main' && params.PUSH
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    retry(3) {
                        sh '''
                            echo "$DH_TOKEN" | docker login \
                                -u "$DH_USER" \
                                --password-stdin

                            docker push $IMAGE:$VERSION
                            docker push $IMAGE:latest
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f webapp-ci-${BUILD_NUMBER} 2>/dev/null || true
                docker network rm ci-net-${BUILD_NUMBER} 2>/dev/null || true
                docker logout || true
            '''
        }

        success {
            echo "✅ Pipeline exécuté, validé et déployé sur la branche ${env.CURRENT_BRANCH} !"
        }

        failure {
            echo "❌ Échec du pipeline sur la branche ${env.CURRENT_BRANCH}"
        }
    }
}