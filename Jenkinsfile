pipeline {
  agent any

  environment {
    NS                = 'registry'
    KUBECONFIG_CRED   = 'kubeconfig'          // Jenkins Credentials: Secret file с kubeconfig
    TG_TOKEN_CRED     = 'telegram-bot-token'  // уже создан у тебя
    TG_CHAT_ID        = '180424264'
    KUBECTL           = 'kubectl'             // если kubectl в PATH у Jenkins
  }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Setup kubecontext') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KCFG')]) {
          sh '''
            set -e
            export KUBECONFIG="$KCFG"
            ${KUBECTL} version --client
            ${KUBECTL} get nodes -o name
          '''
        }
      }
    }

    stage('Apply K8s manifests') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CRED, variable: 'KCFG')]) {
          sh '''
            set -e
            export KUBECONFIG="$KCFG"
            # namespace (идемпотентно)
            ${KUBECTL} apply -f k8s/00-namespace.yaml
            # storage
            ${KUBECTL} apply -f k8s/10-pv.yaml
            ${KUBECTL} apply -f k8s/20-pvc.yaml
            # сам registry
            ${KUBECTL} apply -f k8s/30-deployment.yaml
            ${KUBECTL} apply -f k8s/40-service.yaml || true
            # если есть ingress:
            if [ -f k8s/50-ingress.yaml ]; then ${KUBECTL} apply -f k8s/50-ingress.yaml; fi

            # ждём, пока pod(ы) раскатятся
            ${KUBECTL} -n ${NS} rollout status deploy/registry --timeout=180s
            ${KUBECTL} -n ${NS} get all,pvc,pv
          '''
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: env.TG_TOKEN_CRED, variable: 'TG_TOKEN')]) {
        sh """
          curl -s -X POST -H 'Content-Type: application/json' \
          --data '{\"chat_id\":\"${TG_CHAT_ID}\",\"text\":\"✅ K8s Registry развернут успешно: ${JOB_NAME} #${BUILD_NUMBER}\"}' \
          https://api.telegram.org/bot${TG_TOKEN}/sendMessage || true
        """
      }
    }
    failure {
      withCredentials([string(credentialsId: env.TG_TOKEN_CRED, variable: 'TG_TOKEN')]) {
        sh """
          curl -s -X POST -H 'Content-Type: application/json' \
          --data '{\"chat_id\":\"${TG_CHAT_ID}\",\"text\":\"❌ Ошибка деплоя K8s Registry: ${JOB_NAME} #${BUILD_NUMBER}\"}' \
          https://api.telegram.org/bot${TG_TOKEN}/sendMessage || true
        """
      }
    }
  }
}
