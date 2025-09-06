pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    NS = 'registry'        // namespace, где будет жить реестр
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Setup kubecontext') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh 'kubectl version --client=true'
          sh 'kubectl config current-context || true'
        }
      }
    }

    stage('Apply K8s manifests') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e

            # namespace (если в 00-namespace.yaml уже есть — просто применим)
            kubectl apply -f 00-namespace.yaml

            # основное — в нужный namespace
            kubectl apply -n ${NS} -f 10-pv-pvc-nfs.yaml
            kubectl apply -n ${NS} -f 20-deploy-svc.yaml
            kubectl apply -n ${NS} -f 40-service.yaml

            # опционально: проверить rollout
            kubectl -n ${NS} rollout status deploy/registry --timeout=120s || true
          """
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data "{\"chat_id\":\"180424264\",\"text\":\"✅ Deploy OK: ${JOB_NAME} #${BUILD_NUMBER}\"}" \
            https://api.telegram.org/bot${TG}/sendMessage || true
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data "{\"chat_id\":\"180424264\",\"text\":\"❌ Deploy FAILED: ${JOB_NAME} #${BUILD_NUMBER}\"}" \
            https://api.telegram.org/bot${TG}/sendMessage || true
        '''
      }
    }
  }
}
