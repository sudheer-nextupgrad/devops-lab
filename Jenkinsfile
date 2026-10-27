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
                    docker logs ${CONTAINER}
                    exit 1
                '''
            }
        }
    }

    post {
        success { echo "Build #${BUILD_NUMBER} deployed. App live on port ${PORT}." }
        failure { echo "Build #${BUILD_NUMBER} failed. Console output dekho." }
    }
}
