#!groovy
library identifier: '3scale-toolbox-jenkins@master',
        retriever: modernSCM(
                [$class: 'GitSCMSource',
                 remote: 'https://github.com/rh-integration/3scale-toolbox-jenkins.git',
                 traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait']]])

def service = null

def baseSystemName = "widget"
def accessToken = "23aff03d2f1f92a1e167c7012b752fbe7163242b9e6dee712dbf7b784d89beba"
def targetSystemName = "widget-api"
def targetInstance = "https://${accessToken}@3scale-admin.6dsvl.apps.shared-na46.openshift.opentlc.com"
def privateBaseURL = "http://three-scale-api-3scale-api.apps.shared-na46.openshift.opentlc.com"
def privateBasePath = "/api"
def developerAccountId = "john"
def publicStagingBaseURL = "https://widget-api-3scale-apicast-staging.6dsvl.apps.shared-na46.openshift.opentlc.com"
def publicProductionBaseURL = "https://widget-api-3scale-apicast-production.6dsvl.apps.shared-na46.openshift.opentlc.com"
def disableTlsValidation = true
def secretName = "3scale-toolbox"
def namespace = "jenkins"
def imageName = "quay.io/redhat/3scale-toolbox"

pipeline {

    agent {
        label "master"
    }

    stages {

        stage('First stage') {
            steps {
                script {

                    echo 'Inside first stage'
                }
            }
        }

        stage("Prepare") {
            steps {
                script {

                    service = toolbox.prepareThreescaleService(
                            openapi: [filename: "widget-api/swagger.yaml"],
                            environment: [baseSystemName                : baseSystemName,
                                          privateBaseUrl                : privateBaseURL,
                                          //privateBasePath               : privateBasePath,
                                          targetSystemName              : targetSystemName,
                                          publicStagingWildcardDomain   : publicStagingBaseURL,
                                          publicProductionWildcardDomain: publicProductionBaseURL],
                            toolbox: [openshiftProject: namespace,
                                      destination     : targetInstance,
                                      image           : imageName,
                                      insecure        : disableTlsValidation,
                                      secretName      : secretName],
                            service: [:],
                            applications: [
                                    [name       : "widget-app",
                                     description: "The widget App",
                                     plan       : "widget-unlimited",
                                     account    : developerAccountId]
                            ],
                            applicationPlans: [
                                    [systemName     : "widget-unlimited",
                                     name           : "Widget Unlimited",
                                     defaultPlan    : true,
                                     published      : true,
                                     costPerMonth   : 0.0,
                                     setupFee       : 0.0,
                                     trialPeriodDays: 0]
                            ]
                    )

                    echo "toolbox version = " + service.toolbox.getToolboxVersion()
                }
            }
        }

        stage('List properties') {
            steps {
                script {

                    proxy = service.readProxy("sandbox")

                    proxy.each { entry -> echo "$entry.key -> $entry.value" }
                }
            }
        }


        stage('Read service proxy config') {
            steps {
                script {

                    service.getEnvironment().properties.each { echo "$it.key -> $it.value" }
                }
            }
        }

//        stage("Import OpenAPI") {
//            steps {
//                script {
//
//                    service.importOpenAPI()
//                    echo "Service with system_name ${service.environment.targetSystemName} created !"
//                }
//            }
//        }

        stage("Create an Application Plan") {
            steps {
                script {

                    service.applyApplicationPlans()
                }
            }
        }

        stage("Create an Application") {
            steps {
                script {

                    service.applyApplication()
                }
            }
        }
    }
}
