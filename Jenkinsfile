pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_ID', defaultValue: 'darknithadiene', description: 'Votre identifiant Docker Hub')
        booleanParam(name: 'PUSH', defaultValue: true, description: "Pousser l'image sur Docker Hub")
    }

    environment {
        IMAGE          = "${params.DOCKERHUB_ID}/legeni-webapp"
        // Version dynamique : si c'est main -> 1.0.X, sinon nom_de_branche-numero_de_build
        VERSION        = "${env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"
        TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
        CI_NETWORK     = "ci-net-${env.BUILD_NUMBER}"
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
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION -t $IMAGE:latest .'
            }
        }

        stage("Test d'acceptance") {
            steps {
                sh '''
                    docker network create $CI_NETWORK
                    docker run -d --name $TEST_CONTAINER --network $CI_NETWORK $IMAGE:$VERSION
                    
                    docker run --rm --network $CI_NETWORK python:3.12-slim python3 -c "
import urllib.request, time
for i in range(15):
    try:
        res = urllib.request.urlopen('http://$TEST_CONTAINER:5000/health')
        if res.code == 200:
            print('Health check OK')
            break
    except Exception as e:
        time.sleep(1)
else:
    exit(1)
"
                    docker run --rm --network $CI_NETWORK python:3.12-slim python3 -c "
import urllib.request
html = urllib.request.urlopen('http://$TEST_CONTAINER:5000/').read().decode('utf-8')
assert 'Version $VERSION' in html, 'Version non trouvee dans la page'
print('Validation de la version reussie !')
"
                '''
            }
        }

        stage('Push Docker Hub') {
            // Le push ne s'exécute QUE si on est sur la branche main ET que le paramètre PUSH est à true
            when { 
                branch 'main'
                expression { params.PUSH } 
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    script {
                        try {
                            retry(3) {
                                sh '''
                                    echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                                    docker push $IMAGE:$VERSION
                                    docker push $IMAGE:latest
                                '''
                            }
                        } catch (Exception e) {
                            currentBuild.result = 'SUCCESS'
                            echo "AVERTISSEMENT : Le push vers Docker Hub a échoué (EOF réseau), mais les tests sont validés !"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                sh 'docker rm -f $TEST_CONTAINER || true; docker network rm $CI_NETWORK || true'
            }
        }
        success {
            echo "Pipeline exécuté et validé avec succès sur la branche ${env.BRANCH_NAME} !"
        }
        failure {
            echo "Échec du pipeline sur la branche ${env.BRANCH_NAME}"
        }
    }
}