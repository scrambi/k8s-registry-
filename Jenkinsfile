pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  environment {
    NS = 'registry' // namespace, где живёт реестр
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Setup kubecontext') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            kubectl version --client=true
            kubectl config current-context || true
          '''
        }
      }
    }

    stage('Apply K8s manifests') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh """
            set -e

            # namespace (идемпотентно)
            kubectl apply -f 00-namespace.yaml

            # манифесты в нужном NS
            kubectl apply -n ${NS} -f 10-pv-pvc-nfs.yaml
            kubectl apply -n ${NS} -f 20-deploy-svc.yaml
            kubectl apply -n ${NS} -f 40-service.yaml

            # ждём раскатку
            kubectl -n ${NS} rollout status deploy/registry --timeout=180s || true

            # сохраним NodePort в файл для post-секции
            kubectl -n ${NS} get svc registry \
              -o jsonpath='{.spec.ports[0].nodePort}' > nodeport.txt
          """
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          set -e
          NODEPORT=$(cat nodeport.txt 2>/dev/null || true)
          # возьмём IP первой ноды (при желании можно захардкодить свой)
          NODEIP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[?(@.type=='InternalIP')].address}" 2>/dev/null || echo "<node_ip>")
          TEXT="✅ Deploy OK: ${JOB_NAME} #${BUILD_NUMBER}\\nRegistry: http://${NODEIP}:${NODEPORT}/v2/_catalog"

          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"180424264","text":"${TEXT}"}
          EOF
        '''.stripIndent()
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"180424264","text":"❌ Deploy FAILED: ${JOB_NAME} #${BUILD_NUMBER} (см. логи Jenkins)"}
          EOF
        '''.stripIndent() || true
      }
    }
  }
}
