@Library('dynatrace@master') _

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'ENVIRONMENT', key: 'application', value: 'sockshop'],
      [context: 'CONTEXTLESS', key: 'service', value: 'carts'],
      [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
    ]
  ]
]

pipeline {
  agent {
    label 'maven'
  }
  environment {
    SERVICE_NAME = "carts"
    VERSION = readFile('version').trim()
    ARTEFACT_ID = "sockshop/" + "${env.SERVICE_NAME}"
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/library/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}:${env.VERSION}"
  }
  stages {
    stage('Maven build') {
      steps {
        checkout scm
        container('maven') {
          sh 'mvn -B clean package'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker push ${env.TAG_DEV}"
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "sed -i 's#image: .*#image: ${env.TAG_DEV}#' manifest/carts.yml"
          sh "kubectl -n dev apply -f manifest/carts.yml"
        }
      }
    }
    stage('DT Deploy Event') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("curl") {
          script {
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules,
              customProperties : [
                [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
                [key: 'Git commit', value: "${env.GIT_COMMIT}"]
              ]
            )
          }
        }
      }
    }
    stage('Run health check in dev for master') {
      when {
        expression {
          return env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        sleep 180

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx', 
              resultsDir: "HealthCheck_${env.SERVICE_NAME}",
              serverUrl: "${env.SERVICE_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
          sh "ls -l H*.tlf"            
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }    
    stage('Run health check in dev for release') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' 
        }
      }
      steps {
        echo "Waiting for the service to start..."
        sleep 180

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx', 
              resultsDir: "HealthCheck_${env.SERVICE_NAME}",
              serverUrl: "${env.SERVICE_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
          sh "ls -l H*.tlf"
          echo "output healthcheck"
          sh "cat HealthCheck_carts_result.tlf"              
          echo "output output.txt"
          sh "cat output.txt"              
            
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "about to start Functional Check in Dev"
        container('jmeter') {
          script {
            def status = executeJMeter (
              scriptName: "jmeter/${env.SERVICE_NAME}_load.jmx", 
              resultsDir: "FuncCheck_${env.SERVICE_NAME}",
              serverUrl: "${env.SERVICE_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "FuncCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
 
          echo "output healthcheck"
          sh "cat HealthCheck_carts_result.tlf"              
          echo "output jmeter.log"
          sh "cat jmeter.log"              
          echo "output output.txt"
          sh "cat output.txt"   
         //   if (status != 0) {
         //     currentBuild.result = 'FAILED'
         //     error "Functional check in dev failed."
         //   }
          }
        }
      }
    }
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
          sh "docker push ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'SERVICE_NAME', value: "${env.SERVICE_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}")
          ]
      }
    }
  }
}
