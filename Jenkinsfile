// CI for the platform repo. There's no chart to publish here — the job VALIDATES the manifests so a
// malformed one is caught before Argo CD tries to apply it. kubeconform checks the core Kubernetes
// kinds strictly; CRDs without public schemas (HTTPRoute, Gateway, EnvoyProxy, PrometheusRule,
// PodMonitor, Instrumentation, Argo CD Applications) are skipped, and Helm values files + raw
// dashboard JSON are ignored (they aren't Kubernetes objects).
pipeline {
  agent any
  triggers { pollSCM('H/2 * * * *') }

  stages {
    stage('Install kubeconform') {
      when { branch 'master' }
      steps {
        sh '''
          if ! command -v kubeconform >/dev/null 2>&1 && [ ! -x ./kubeconform ]; then
            curl -fsSL https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-linux-amd64.tar.gz | tar xz kubeconform
            chmod +x ./kubeconform
          fi
        '''
        script {
          env.KC = sh(script: 'command -v kubeconform >/dev/null 2>&1 && echo kubeconform || echo ./kubeconform', returnStdout: true).trim()
        }
      }
    }

    stage('Validate manifests') {
      when { branch 'master' }
      steps {
        sh "${env.KC} -strict -ignore-missing-schemas -summary -ignore-filename-pattern '.*values\\.yaml' -ignore-filename-pattern '.*\\.json' gateway monitoring observability headlamp network-policies metrics-server vpa"
      }
    }
  }

  post {
    failure { echo 'Manifest validation failed - check the stage log.' }
  }
}
