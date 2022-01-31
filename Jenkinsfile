#!groovy
library identifier: '3scale-toolbox-jenkins@master',
        retriever: modernSCM(
                [$class: 'GitSCMSource',
                 remote: 'https://github.com/rh-integration/3scale-toolbox-jenkins.git',
                 traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait']]])

def service = null

def targetSystemName = "widget-system"
def targetInstance = "3scale-on-prem"
def privateBaseURL = "http://three-scale-api-3scale-api.apps.shared-na46.openshift.opentlc.com:80/api/"
def testUserKey = "97daa2af299f93316d83331b99a96429"
def developerAccountId = "admin"
def publicStagingBaseURL = "https://widget-api-3scale-apicast-staging.6dsvl.apps.shared-na46.openshift.opentlc.com:443"
def publicProductionBaseURL = "https://widget-api-3scale-apicast-production.6dsvl.apps.shared-na46.openshift.opentlc.com:443"
def disableTlsValidation = true
def secretName = "3scale-toolbox"
def namespace = "jenkins"

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
                            environment: [baseSystemName                : "widget",
                                          privateBaseUrl                : privateBaseURL,
                                          privateBasePath               : "/api/",
                                          publicStagingWildcardDomain   : publicStagingBaseURL,
                                          publicProductionWildcardDomain: publicProductionBaseURL],
                            toolbox: [openshiftProject: namespace,
                                      destination     : targetInstance,
                                      image           : "quay.io/redhat/3scale-toolbox",
                                      insecure        : disableTlsValidation,
                                      secretName      : secretName],
                            service: [:],
                            applications: [
                                    [name: "Widget Application", description: "The widget App", plan: "Widget Unlimited", account: developerAccountId]
                            ],
                            applicationPlans: [
                                    [systemName: "widget-unlimited", name: "Widget Unlimited", defaultPlan: true, published: true],
                            ]
                    )

                    echo "toolbox version = " + service.toolbox.getToolboxVersion()
                }
            }
        }
        stage('Provision config map') {
            steps {
                script {

                    echo "environment" + service.getEnvironment()
                }
            }
        }
    }
}
