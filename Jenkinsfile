pipeline {
  agent any

  environment {
    APP = "polling-app"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set target environment') {
      steps {
        script {
          if (!env.BRANCH_NAME) {
            env.BRANCH_NAME = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
          }
          echo "Detected branch: ${env.BRANCH_NAME}"

          if (env.BRANCH_NAME == 'dev') {
            env.TARGET = "clusters/dev"
            env.KUBECONFIG_CRED = "kubeconfig-dev"
          } else if (env.BRANCH_NAME == 'prod') {
            env.TARGET = "clusters/prod"
            env.KUBECONFIG_CRED = "kubeconfig-prod"
          } else {
            error "Unsupported branch: ${env.BRANCH_NAME}. Must be dev or prod."
          }
          echo "Target set to: ${env.TARGET}"
        }
      }
    }

    stage('Deploy using Kustomize') {
      steps {
        withCredentials([file(credentialsId: "${env.KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG="${KUBECONFIG_FILE}"
            echo "Deploying ${APP} using overlay ${TARGET}"
            kubectl version --client
            kubectl config current-context
            kubectl apply -k ${TARGET}
            kubectl rollout status deployment/${APP} --timeout=60s || true
            kubectl get deploy,svc -l app=${APP} -o wide
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Successfully deployed ${APP} to ${env.BRANCH_NAME} environment."
    }
    failure {
      echo "Deployment failed for ${env.BRANCH_NAME}."
    }
  }
}
