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

    stage('Set target') {
      steps {
        script {
          // BRANCH_NAME is set by multibranch pipeline or you can use env.GIT_BRANCH
          if (env.BRANCH_NAME == null) {
            // fallback for plain pipeline runs
            env.BRANCH_NAME = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
          }
          echo "Branch detected: ${env.BRANCH_NAME}"
          if (env.BRANCH_NAME == 'dev') {
            env.TARGET = "clusters/dev"
            env.KUBECONFIG_CRED = "kubeconfig-dev" // Jenkins secret file id for dev kubeconfig
          } else if (env.BRANCH_NAME == 'prod') {
            env.TARGET = "clusters/prod"
            env.KUBECONFIG_CRED = "kubeconfig-prod" // Jenkins secret file id for prod kubeconfig
          } else {
            error "Unsupported branch: ${env.BRANCH_NAME}. Use dev or prod."
          }
          echo "Target overlay: ${env.TARGET}"
        }
      }
    }

    stage('Apply kustomize') {
      steps {
        // stash/unstash not needed; using credentials file to kubectl
        withCredentials([file(credentialsId: "${env.KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG="${KUBECONFIG_FILE}"
            echo "Using kubeconfig: $KUBECONFIG"
            kubectl version --client
            # optional: show current context for debugging
            kubectl config current-context
            # perform kustomize apply
            kubectl apply -k ${TARGET}
            # wait rollout (optional)
            kubectl rollout status deployment/polling-app --namespace default --timeout=120s || true
            # show resources
            kubectl get deploy,svc -l app=polling-app -o wide
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Deployment to ${env.BRANCH_NAME} successful."
    }
    failure {
      echo "Deployment failed."
    }
  }
}
