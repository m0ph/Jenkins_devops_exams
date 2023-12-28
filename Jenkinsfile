pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "m0ph" // replace this with your docker-id
DOCKER_IMAGE_CAST = "jenkinsexamcast"
DOCKER_IMAGE_MOVIE = "jenkinsexammovie"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any
stages {
    stage('Docker Build Images'){
        steps {
            script {
                sh '''
                docker rm -f $DOCKER_IMAGE_MOVIE
                docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
                sleep 6
                '''
            }
            script {
                sh '''
                docker rm -f $DOCKER_IMAGE_CAST
                docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                sleep 6
                '''
            }
        }
    }
    stage('Docker run'){
        steps {
            script {
                sh '''
                docker run -d --name cast_db --env POSTGRES_USER=cast_db_username --env POSTGRES_PASSWORD=cast_db_password --env POSTGRES_DB=cast_db_dev postgres:12.1-alpine
                docker run -d --name movie_db --env POSTGRES_USER=movie_db_username --env POSTGRES_PASSWORD=movie_db_password --env POSTGRES_DB=movie_db_dev postgres:12.1-alpine
                sleep 10
                docker run -d --name cast_service -p 8002:8000 --rm --env DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                sleep 5
                docker run -d --name movie-service -p 8001:8000 --rm --env DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev --env CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                sleep 5
                docker run -d --name nginx -p 8080:8080 --volume ./nginx_config.conf:/etc/nginx/conf.d/default.conf nginx:latest
                '''
            }
        }
    }
    stage('Test Connection'){
        steps {
            script {
                sh '''
                docker run --rm curlimages/curl:latest -L -v http://localhost:8001
                docker run --rm curlimages/curl:latest -L -v http://localhost:8002
                docker run --rm curlimages/curl:latest -L -v http://movie_service:8000
                docker run --rm curlimages/curl:latest -L -v http://cast_service:8000
                '''
            }
        }
    }
    stage('Docker push')
}