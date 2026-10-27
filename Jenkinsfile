pipeline {
    agent any

    environment {
        IMAGE     = 'devops-lab'
        CONTAINER = 'devops-lab'
        PORT      = '5000'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'rm -f .prev_image'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t ${IMAGE}:${BUILD_NUMBER} -t ${IMAGE}:latest .'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run --rm ${IMAGE}:${BUILD_NUMBER} python -c "import app"'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    PREV=$(docker inspect -f '{{.Config.Image}}' ${CONTAINER} 2>/dev/null || true)
                    echo "previous image: ${PREV:-none}"
                    echo "${PREV}" > .prev_image
                    docker rm -f ${CONTAINER} || true
                    docker run -d --name ${CONTAINER} -p ${PORT}:5000 --restart unless-stopped ${IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    for i in $(seq 1 15); do
                        STATUS=$(docker inspect -f '{{.State.Health.Status}}' ${CONTAINER})
                        echo "health: $STATUS"
                        if [ "$STATUS" = "healthy" ]; then exit 0; fi
                        sleep 5
                    done
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "Build #${BUILD_NUMBER} deployed. App live on port ${PORT}."
            sh '''
                docker images ${IMAGE} --format '{{.Tag}}' | grep -E '^[0-9]+$' | sort -nr | tail -n +6 \
                  | xargs -r -I{} docker rmi ${IMAGE}:{} || true
            '''
        }
        failure {
            echo "Build #${BUILD_NUMBER} failed."
            sh '''
                if [ ! -f .prev_image ]; then
                    echo "Failed before deploy, running app untouched."
                    exit 0
                fi
                PREV=$(cat .prev_image)
                echo "--- failed container logs ---"
                docker logs --tail 30 ${CONTAINER} || true
                docker rm -f ${CONTAINER} || true
                if [ -n "$PREV" ]; then
                    echo "ROLLBACK to $PREV"
                    docker run -d --name ${CONTAINER} -p ${PORT}:5000 --restart unless-stopped $PREV
                else
                    echo "No previous version to roll back to."
                fi
            '''
        }
    }
}
