def deployEnv(String ns) {
  withCredentials([
    usernamePassword(credentialsId: 'MOVIE_DB_CREDENTIALS', usernameVariable: 'MOVIE_DB_USER', passwordVariable: 'MOVIE_DB_PASS'),
    usernamePassword(credentialsId: 'CAST_DB_CREDENTIALS',  usernameVariable: 'CAST_DB_USER',  passwordVariable: 'CAST_DB_PASS'),
    file(credentialsId: 'KUBECONFIG', variable: 'KUBECONFIG')
  ]) {
    sh """
      set -eu
      chmod 600 "\$KUBECONFIG" || true
      export KUBECONFIG="\$KUBECONFIG"
      kubectl create namespace ${ns} --dry-run=client -o yaml | kubectl apply -f -

      kubectl -n ${ns} create secret generic movie-db-secret \\
        --from-literal=username=\"${MOVIE_DB_USER}\" \\
        --from-literal=password=\"${MOVIE_DB_PASS}\" \\
        --dry-run=client -o yaml | kubectl apply -f -

      kubectl -n ${ns} create secret generic cast-db-secret \\
        --from-literal=username=\"${CAST_DB_USER}\" \\
        --from-literal=password=\"${CAST_DB_PASS}\" \\
        --dry-run=client -o yaml | kubectl apply -f -

      helm upgrade --install app deploy/k8s/helm/app \\
        -n ${ns} -f deploy/k8s/helm/values/${ns}.yaml \\
        --set-string movie-service.image.tag=\"${GIT_COMMIT}\" \\
        --set-string cast-service.image.tag=\"${GIT_COMMIT}\" \\
        --set-string movie-service.postgresql.existingSecret=\"movie-db-secret\" \\
        --set-string cast-service.postgresql.existingSecret=\"cast-db-secret\" \\
        --dependency-update \\
        --atomic --timeout 5m --history-max 5
    """
  }
}

pipeline {
  agent any

  environment {
    DOCKER_ID = "mindrelay"
    MOVIE_IMG = "${DOCKER_ID}/movie-service"
    CAST_IMG  = "${DOCKER_ID}/cast-service"
  }


  stages {
    stage('build') {
      environment {
        DOCKERHUB_TOKEN = credentials("DOCKERHUB_TOKEN")
      }
      when {
        anyOf {
          changeRequest(target: 'develop')
          changeRequest(target: 'master')
          branch 'develop'
          branch 'master'
        }
      }
      steps {
        sh '''
          set -eu
          echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKER_ID" --password-stdin
  
          docker build -f movie-service/Dockerfile -t "$MOVIE_IMG:$GIT_COMMIT" ./movie-service
          docker build -f cast-service/Dockerfile -t "$CAST_IMG:$GIT_COMMIT" ./cast-service

          docker push "mindrelay/movie-service:$GIT_COMMIT"
          docker push "mindrelay/cast-service:$GIT_COMMIT"
        '''
      }
    }

    stage('run') {
      when {
        anyOf {
          changeRequest(target: 'develop')
          changeRequest(target: 'master')
          branch 'develop'
          branch 'master'
        }
      }
      steps {
        sh '''
          set -eu
          docker compose -p ci -f deploy/docker-compose.ci.yml pull
          docker compose -p ci -f deploy/docker-compose.ci.yml up -d

          for i in $(seq 1 15); do
            docker compose -p ci -f deploy/docker-compose.ci.yml run --rm --no-deps \
              curl "curl -fsS http://nginx:8080/api/v1/movies/docs" >/dev/null &&
            docker compose -p ci -f deploy/docker-compose.ci.yml run --rm --no-deps \
              curl "curl -fsS http://nginx:8080/api/v1/casts/docs" >/dev/null &&
            exit 0
            sleep 1
          done

          docker compose -p ci -f deploy/docker-compose.ci.yml logs --tail=200
          exit 1
        '''
      }
      post {
        always {
          sh 'docker compose -p ci -f deploy/docker-compose.ci.yml down -v || true'
        }
      }
    }

    stage('deploy_dev') {
      when { branch 'develop' }
      steps {
        script {
          deployEnv('dev')
        }
      }
    }

    stage('deploy_qa') {
      when { branch 'develop' }
      steps {
        script {
          deployEnv('qa')
        }
      }
    }

    stage('deploy_staging') {
      when { branch 'develop' }
      steps {
        script {
          deployEnv('staging')
        }
      }
    }

    stage('deploy_prod') {
      when { branch 'master' }
      steps {
        input message: 'Run deploy'
        script {
          deployEnv('prod')
        }
      }
    }
  }
}
