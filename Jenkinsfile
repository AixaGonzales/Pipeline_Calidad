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
      steps { bat 'mvn -B clean verify' }
    }
    stage('Análisis SonarQube') {
      steps {
        withSonarQubeEnv('SonarQube') {
          bat 'mvn -B sonar:sonar -Dsonar.projectKey=psw-pipeline-base -Dsonar.projectName="PSW Pipeline Base"'
        }
      }
    }
    stage('Quality Gate') {
      steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
    }
    stage('Pruebas JMeter') {
      steps {
        powershell '''
          $ErrorActionPreference = 'Stop'
          New-Item -ItemType Directory -Force -Path 'target/jmeter' | Out-Null
          $app = Start-Process -FilePath 'java' `
            -ArgumentList @('-jar', 'target/psw-pipeline-base-0.0.1-SNAPSHOT.jar', "--server.port=$env:APP_PORT") `
            -PassThru `
            -RedirectStandardOutput 'target/app.log' `
            -RedirectStandardError 'target/app-error.log'
          try {
            $ready = $false
            for ($attempt = 0; $attempt -lt 60; $attempt++) {
              try {
                Invoke-WebRequest -Uri "http://127.0.0.1:$env:APP_PORT/products" -UseBasicParsing | Out-Null
                $ready = $true
                break
              } catch {
                Start-Sleep -Seconds 2
              }
            }
            if (-not $ready) { throw 'La aplicación no respondió en /products dentro del tiempo esperado.' }
            & jmeter -n -t $env:JMETER_PLAN `
              "-JbaseUrl=http://127.0.0.1:$env:APP_PORT" `
              -l target/jmeter/results.jtl -e -o target/jmeter/report
            if ($LASTEXITCODE -ne 0) { throw "JMeter terminó con código $LASTEXITCODE." }
          } finally {
            if ($app -and -not $app.HasExited) { Stop-Process -Id $app.Id -Force }
          }
        '''
      }
      post { always { archiveArtifacts artifacts: 'target/jmeter/**, target/app.log, target/app-error.log', allowEmptyArchive: true } }
    }
  }
  post {
    success { echo "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} finalizó correctamente: ${env.BUILD_URL}" }
    failure { echo "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} presentó un error: ${env.BUILD_URL}" }
    unstable { echo "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} quedó inestable: ${env.BUILD_URL}" }
  }
}
