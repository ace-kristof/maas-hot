@Library('ace@master') _ 

def tagMatchRules = [
  [
    "meTypes": [
      ["meType": "SERVICE"]
    ],
    tags : [
      ["context": "CONTEXTLESS", "key": "app", "value": "simplenodeservice"],
      ["context": "CONTEXTLESS", "key": "environment", "value": "staging"]
    ]
  ]
]

pipeline {
    parameters {
        string(name: 'APP_NAME', defaultValue: 'simplenodeservice', description: 'The name of the service to deploy.', trim: true)
    }
    agent {
        label 'kubegit'
    }

    stages {
        stage('DT send test start event') {
          steps {
               container("curl") {
                    script {
                     def status = pushDynatraceInfoEvent (
                            tagRule : tagMatchRules,
                           source: "Jmeter",
                            description: "Performance test for ${env.APP_NAME}",
                            title: "Jmeter Start",
                            deploymentVersion: "${env.BUILD}",
                          customProperties : [
                              [key: 'VU Count', value: "1"],
                              [key: 'Loop Count', value: "10"]
                          ]
                       )
                  }
             }
          }
        }
        stage('Run performance test') {
            steps {
                checkout scm
                container('jmeter') {
                    script {
                    def status = executeJMeter ( 
                        scriptName: "jmeter/simplenodeservice_load.jmx",
                        resultsDir: "perfCheck_${env.APP_NAME}_staging_${BUILD_NUMBER}",
                        serverUrl: "simplenodeservice.staging", 
                        serverPort: 80,
                        checkPath: '/health',
                        vuCount: 1,
                        loopCount: 10,
                        LTN: "perfCheck_${env.APP_NAME}_${BUILD_NUMBER}",
                        funcValidation: false,
                        avgRtValidation: 4000
                    )
                    if (status != 0) {
                        currentBuild.result = 'FAILED'
                        error "Performance test in staging failed."
                    }
                    }
                }
            }
        }
        stage('DT send test stop event') {
          steps {
               container("curl") {
                    script {
                     def status = pushDynatraceInfoEvent (
                            tagRule : tagMatchRules,
                           source: "Jmeter",
                            description: "Performance test for ${env.APP_NAME}",
                            title: "Jmeter Start",
                            deploymentVersion: "${env.BUILD}",
                          customProperties : [
                              [key: 'VU Count', value: "1"],
                              [key: 'Loop Count', value: "10"]
                          ]
                       )
                  }
             }
          }
        }
        stage('Manual approval') {
            // no agent, so executors are not used up when waiting for approvals
            agent none
            steps {
                script {
                    try {
                        timeout(time:10, unit:'MINUTES') {
                            env.APPROVE_PROD = input message: 'Promote to Production', ok: 'Continue', parameters: [choice(name: 'APPROVE_PROD', choices: 'YES\nNO', description: 'Deploy from STAGING to PRODUCTION?')]
                            if (env.APPROVE_PROD == 'YES'){
                                env.DPROD = true
                            } else {
                                env.DPROD = false
                            }
                        }
                    } catch (error) {
                        env.DPROD = true
                        echo 'Timeout has been reached! Deploy to PRODUCTION automatically activated'
                    }
                }
            }
        }

        stage('Promote to production') {
            // no agent, so executors are not used up when waiting for other job to complete
            agent none
            when {
                expression {
                    return env.DPROD == 'true'
                }
            }
            steps {
                build job: "4. Deploy production",
                parameters: [
                    string(name: 'APP_NAME', value: "${env.APP_NAME}")
                ]
            }
        }  
    }
}
