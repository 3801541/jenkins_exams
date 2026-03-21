pipeline {
  environment {
    DOCKER_ID = "ninine"
    DOCKER_MOVIE_IMAGE = "movie-service"
    BUILD_MOVIE_CONTEXT = "movie-service"
    BUILD_CAST_CONTEXT = "cast-service"
    DOCKER_CAST_IMAGE = "cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0"
    POSTGRES_MOVIE_USER = "movie_db_username"
    POSTGRES_MOVIE_PASSWORD = "movie_db_password"
    POSTGRES_MOVIE_DB = "movie_db_dev"
    POSTGRES_CAST_USER = "cast_db_username"
    POSTGRES_CAST_PASSWORD = "cast_db_password"
    POSTGRES_CAST_DB = "cast_db_dev"
    DATABASE_MOVIE_URI = "postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev"
    CAST_SERVICE_HOST_URL = "http://cast_service:8000/api/v1/casts/"
    DATABASE_CAST_URI = "postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev"
  }
  agent any
  stages {
    stage('Docker Build') {
      steps {
        script {
          sh '''
            docker rm -f $(docker ps -aq) || true
            docker build -t "$DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG" -f "$BUILD_MOVIE_CONTEXT/Dockerfile" "$BUILD_MOVIE_CONTEXT"
            docker build -t "$DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG" -f "$BUILD_CAST_CONTEXT/Dockerfile" "$BUILD_CAST_CONTEXT"
            sleep 6
          '''
        }
      }
    }
    stage('Docker run') {
      steps {
        script {
          sh '''
            docker network create backend || true
            docker run -d --name movie_db --network backend -e POSTGRES_USER=$POSTGRES_MOVIE_USER -e POSTGRES_PASSWORD=$POSTGRES_MOVIE_PASSWORD -e POSTGRES_DB=$POSTGRES_MOVIE_DB postgres:12.1-alpine
            docker run -d --name cast_db --network backend -e POSTGRES_USER=$POSTGRES_CAST_USER -e POSTGRES_PASSWORD=$POSTGRES_CAST_PASSWORD -e POSTGRES_DB=$POSTGRES_CAST_DB postgres:12.1-alpine
            docker run -d --name movie_service --network backend -p 8001:8000 -e DATABASE_URI=$DATABASE_MOVIE_URI -e CAST_SERVICE_HOST_URL=$CAST_SERVICE_HOST_URL $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
            docker run -d --name cast_service --network backend -p 8002:8000 -e DATABASE_URI=$DATABASE_CAST_URI $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
            docker run -d --name nginx --network backend -p 8080:8080 -v $(pwd)/nginx_config.conf:/etc/nginx/conf.d/default.conf nginx:latest
            sleep 10
          '''
        }
      }
    }
    stage('Test Acceptance') {
      steps {
        script {
          sh '''
            curl http://localhost:8080/api/v1/movies/docs
            curl http://localhost:8080/api/v1/casts/docs
          '''
        }
      }
    }
    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
      }
      steps {
        script {
          sh '''
            docker login -u $DOCKER_ID -p $DOCKER_PASS
            docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
            docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
          '''
        }
      }
    }
  }
  post {
    failure {
      echo "This will run if the job failed"
      mail to: "nsitang1@gmail.com",
        subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
        body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
    }
  }
}
