pipeline {
  agent any
  options { timestamps(); disableConcurrentBuilds() }
  environment {
    APP_PORT = '8080'
    JMETER_PLAN = 'jmeter/pipeline-calidad.jmx'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build y pruebas') {
      steps { sh 'mvn -B clean verify' }
    }
    stage('Análisis SonarQube') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'mvn -B sonar:sonar -Dsonar.projectKey=psw-pipeline-base -Dsonar.projectName="PSW Pipeline Base"'
        }
      }
    }
    stage('Quality Gate') {
      steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
    }
    stage('Pruebas JMeter') {
      steps {
        sh '''
          set -eu
          mkdir -p target/jmeter
          java -jar target/psw-pipeline-base-0.0.1-SNAPSHOT.jar --server.port=${APP_PORT} > target/app.log 2>&1 &
          APP_PID=$!
          trap 'kill "$APP_PID" 2>/dev/null || true' EXIT
          for attempt in $(seq 1 60); do
            if curl -fsS "http://127.0.0.1:${APP_PORT}/products" >/dev/null; then break; fi
            sleep 2
          done
          curl -fsS "http://127.0.0.1:${APP_PORT}/products" >/dev/null
          jmeter -n -t "${JMETER_PLAN}" -JbaseUrl="http://127.0.0.1:${APP_PORT}" \
            -l target/jmeter/results.jtl -e -o target/jmeter/report
        '''
      }
      post { always { archiveArtifacts artifacts: 'target/jmeter/**, target/app.log', allowEmptyArchive: true } }
    }
  }
  post {
    success { slackSend channel: '#pipeline-calidad', color: 'good', message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} finalizó correctamente: ${env.BUILD_URL}" }
    failure { slackSend channel: '#pipeline-calidad', color: 'danger', message: "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} presentó un error: ${env.BUILD_URL}" }
    unstable { slackSend channel: '#pipeline-calidad', color: 'warning', message: "⚠️ ${env.JOB_NAME} #${env.BUILD_NUMBER} quedó inestable: ${env.BUILD_URL}" }
  }
}
