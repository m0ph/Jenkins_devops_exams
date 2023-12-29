pipeline {
environment { 
DOCKER_ID = "m0ph" 
DOCKER_IMAGE_CAST = "jenkinsexamcast"
DOCKER_IMAGE_MOVIE = "jenkinsexammovie"
DOCKER_TAG = "v.${BUILD_ID}.0"
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
                docker run -d --name cast_db --rm --network test_network --env POSTGRES_USER=cast_db_username --env POSTGRES_PASSWORD=cast_db_password --env POSTGRES_DB=cast_db_dev postgres:12.1-alpine
                docker run -d --name movie_db --rm --network test_network --env POSTGRES_USER=movie_db_username --env POSTGRES_PASSWORD=movie_db_password --env POSTGRES_DB=movie_db_dev postgres:12.1-alpine
                sleep 10
                docker run -d --name cast_service --rm --network test_network --env DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                sleep 5
                docker run -d --name movie_service --rm --network test_network --env DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev --env CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                sleep 5
                docker run -d --name nginx --rm --network test_network --volume ./nginx_config.conf:/etc/nginx/conf.d/default.conf nginx:latest
                '''
            }
        }
    }
    stage('Test Connection'){
        steps {
            script {
                sh '''
                docker run --rm --network test_network curlimages/curl:latest -L -v http://movie_service:8000
                docker run --rm --network test_network curlimages/curl:latest -L -v http://cast_service:8000
                docker run --rm --network test_network curlimages/curl:latest -L -v http://nginx:8080/api/v1/movies
                docker run --rm --network test_network curlimages/curl:latest -L -v http://nginx:8080/api/v1/casts
                '''
            }
        }
    }
    stage('Docker push'){
        environment {
            DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        }
        steps {
            script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                '''
            }
        }
    }
    stage('Deploy to dev'){
        environment {
            KUBECONFIG = credentials("config")
        }
        steps {
            script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp examjenkins/values.yaml values.yml
                cat values.yml
                sed -i "s+tag:.*replace.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app examjenkins --values=values.yml --namespace dev
                '''
            }
        }
    }
    stage('Deploy to QA'){
        environment {
            KUBECONFIG = credentials("config")
        }
        steps {
            script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp examjenkins/values.yaml values.yml
                cat values.yml
                sed -i "s+tag:.*replace.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app examjenkins --values=values.yml --namespace qa
                '''
            }
        }
    }
    stage('Deploy to Staging'){
        environment {
            KUBECONFIG = credentials("config")
        }
        steps {
            script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp examjenkins/values.yaml values.yml
                cat values.yml
                sed -i "s+tag:.*replace.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app examjenkins --values=values.yml --namespace staging
                '''
            }
        }
    }
    stage('Deploy to Production'){
        environment {
            KUBECONFIG = credentials("config")
        }
        steps {
            timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }
            script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp examjenkins/values.yaml values.yml
                cat values.yml
                sed -i "s+tag:.*replace.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app examjenkins --values=values.yml --namespace prod
                '''
            }
        }
    }
    post {
        failure {
            sh '''
            docker stop $(docker ps -a -q)
            '''
        }
        script {
            sh '''
            docker stop $(docker ps -a -q)
            '''
        }
    }
}
}