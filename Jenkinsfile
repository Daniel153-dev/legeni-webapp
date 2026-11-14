pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKERHUB_ID',
            defaultValue: 'darknithadiene',
            description: 'Votre identifiant Docker Hub'
        )

        booleanParam(
            name: 'PUSH',
            defaultValue: true,
            description: "Pousser l'image sur Docker Hub"
        )
    }

    environment {
        IMAGE = "${params.DOCKERHUB_ID}/legeni-webapp"

        VERSION = "${env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"

        TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
        CI_NETWORK = "ci-net-${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

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

        stage('Push Docker Hub') {
            when {
                branch 'main'
                expression {
                    return params.PUSH
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
                docker rm -f $TEST_CONTAINER || true
                docker network rm $CI_NETWORK || true
                docker logout || true
            '''
        }

        success {
            echo "Pipeline exécuté et validé sur la branche ${env.BRANCH_NAME} !"
        }

        failure {
            echo "Échec du pipeline sur la branche ${env.BRANCH_NAME}"
        }
    }
}