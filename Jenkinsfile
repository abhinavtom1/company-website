
pipeline {

    agent none

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKERHUB_USERNAME = 'abhinavtom'
        DOCKERHUB_REPOSITORY = 'company-website'
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub-credentials'

        GITHUB_REPOSITORY_URL =
            'https://github.com/abhinavtom1/company-website.git'

        IMAGE_NAME = 'abhinavtom/company-website'
        CONTAINER_NAME = 'company-website'
        APP_PORT = '9090'
    }

    stages {

        // ==========================================
        // PART 1: BUILD SERVER
        // ==========================================

        stage('Build Server Tasks') {

            agent {
                label 'build-server'
            }

            stages {

                stage('Checkout') {
                    steps {
                        echo 'Checking out source code from GitHub'

                        git branch: 'main',
                            url: "${GITHUB_REPOSITORY_URL}"
                    }
                }

                stage('Docker Image Build') {
                    steps {
                        echo "Building Docker image version ${BUILD_NUMBER}"

                        sh '''
                            docker build \
                                -t $IMAGE_NAME:$BUILD_NUMBER .
                        '''
                    }
                }

                stage('Docker Hub Login') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: DOCKERHUB_CREDENTIAL_ID,
                                usernameVariable: 'DH_USER',
                                passwordVariable: 'DH_PASS'
                            )
                        ]) {
                            sh '''
                                echo "$DH_PASS" | docker login \
                                    -u "$DH_USER" --password-stdin
                            '''
                        }
                    }
                }

                stage('Docker Image Push') {
                    steps {
                        echo "Pushing image version ${BUILD_NUMBER}"

                        sh '''
                            docker push $IMAGE_NAME:$BUILD_NUMBER
                        '''
                    }
                }
            }

            post {
                always {
                    sh 'docker logout || true'
                }
            }
        }

        // ==========================================
        // PART 2: DEPLOYMENT AGENT
        // ==========================================

        stage('Deployment Server Tasks') {

            agent {
                label 'deployment-agent'
            }

            stages {

                stage('Docker Hub Login') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: DOCKERHUB_CREDENTIAL_ID,
                                usernameVariable: 'DH_USER',
                                passwordVariable: 'DH_PASS'
                            )
                        ]) {
                            sh '''
                                echo "$DH_PASS" | docker login \
                                    -u "$DH_USER" --password-stdin
                            '''
                        }
                    }
                }

                stage('Pull Docker Image') {
                    steps {
                        echo "Pulling image version ${BUILD_NUMBER}"

                        sh '''
                            docker pull $IMAGE_NAME:$BUILD_NUMBER
                        '''
                    }
                }

                stage('Stop Old Container') {
                    steps {
                        echo 'Removing old website container'

                        sh '''
                            docker rm -f $CONTAINER_NAME || true
                        '''
                    }
                }

                stage('Run New Container') {
                    steps {
                        echo 'Starting new website container'

                        sh '''
                            docker run -d \
                                --name $CONTAINER_NAME \
                                --restart unless-stopped \
                                -p $APP_PORT:80 \
                                $IMAGE_NAME:$BUILD_NUMBER
                        '''
                    }
                }

                stage('Verify Deployment') {
                    steps {
                        echo 'Checking website deployment'

                        sh '''
                            sleep 3

                            docker ps --filter "name=$CONTAINER_NAME"

                            for i in 1 2 3 4 5; do
                                if curl -fsS \
                                    http://localhost:$APP_PORT/ \
                                    -o /dev/null; then

                                    echo "Website is running successfully!"
                                    exit 0
                                fi

                                sleep 2
                            done

                            echo "Website deployment verification failed"

                            docker logs $CONTAINER_NAME

                            exit 1
                        '''
                    }
                }
            }

            post {
                always {
                    sh 'docker logout || true'
                }
            }
        }
    }

    // ==========================================
    // POST BUILD ACTIONS
    // ==========================================

    post {
        success {
            echo """
            ======================================
            CI/CD PIPELINE SUCCESSFUL!
            ======================================
            Docker Image: ${IMAGE_NAME}:${BUILD_NUMBER}
            Container: ${CONTAINER_NAME}
            Application Port: ${APP_PORT}
            ======================================
            """
        }

        failure {
            echo 'CI/CD PIPELINE FAILED!'
            echo 'Check Console Output for the failed stage.'
        }

        always {
            echo "Pipeline execution completed: Build #${BUILD_NUMBER}"
        }
    }
}
