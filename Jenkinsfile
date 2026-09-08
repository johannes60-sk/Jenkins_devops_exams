pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        MOVIE_IMAGE = 'johannes60/jenkins-movie-service'
        CAST_IMAGE  = 'johannes60/jenkins-cast-service'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Tests') {
            steps {
                sh 'python3 -m compileall -q movie-service cast-service'
                sh 'helm lint charts'
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    docker build -t "$MOVIE_IMAGE:$BUILD_NUMBER" movie-service
                    docker build -t "$CAST_IMAGE:$BUILD_NUMBER" cast-service
                '''
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$MOVIE_IMAGE:$BUILD_NUMBER"
                        docker push "$CAST_IMAGE:$BUILD_NUMBER"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Dev') {
            steps {
                sh '''
                    helm upgrade --install jenkins-dev charts \
                      --namespace dev \
                      --set images.movie.tag="$BUILD_NUMBER" \
                      --set images.cast.tag="$BUILD_NUMBER" \
                      --wait --timeout 5m

                    helm test jenkins-dev -n dev --logs
                '''
            }
        }

        stage('Deploy QA') {
            steps {
                sh '''
                    helm upgrade --install jenkins-qa charts \
                      --namespace qa \
                      --set images.movie.tag="$BUILD_NUMBER" \
                      --set images.cast.tag="$BUILD_NUMBER" \
                      --wait --timeout 5m

                    helm test jenkins-qa -n qa --logs
                '''
            }
        }

        stage('Deploy Staging') {
            steps {
                sh '''
                    helm upgrade --install jenkins-staging charts \
                      --namespace staging \
                      --set images.movie.tag="$BUILD_NUMBER" \
                      --set images.cast.tag="$BUILD_NUMBER" \
                      --wait --timeout 5m

                    helm test jenkins-staging -n staging --logs
                '''
            }
        }

        stage('Deploy Production') {
            when {
                branch 'master'
            }
            input {
                message 'Déployer cette version en production ?'
                ok 'Déployer'
            }
            steps {
                sh '''
                    helm upgrade --install jenkins-prod charts \
                      --namespace prod \
                      --set images.movie.tag="$BUILD_NUMBER" \
                      --set images.cast.tag="$BUILD_NUMBER" \
                      --wait --timeout 5m

                    helm test jenkins-prod -n prod --logs
                '''
            }
        }
    }
}
