#!groovy

IMAGE_NAME = "quay.io/redhat/3scale-toolbox"

List APPLICATIONS
List APPLICATION_PLANS
List BACKENDS

String jobId() {

    strLength = 5
    source = "abcdefghijklmnopqrstuvwxyz0123456789"
    random = new Random()

    builder = new StringBuilder(strLength)
    for (int i = 0; i < strLength; i++) {
        builder.append(source[random.nextInt(source.length() - 1)])
    }

    return "job-${env.APP_BASE_NAME}-" + builder.toString().toLowerCase()
}

String waitForPod(String prefix) {

    podName = ""
    startsWith:
    while (!podName.startsWith(prefix)) {
        sleep 1
        pods = openshift.raw("get pods -o custom-columns=POD:.metadata.name --no-headers").out
        for (String item : pods.split("\n")) {
            if (item.startsWith(prefix)) {
                podName = item
                break startsWith
            }
        }
    }

    return podName
}

void waitForSuccessfulCompletion(String pod) {

    phase = ""
    while (!"Succeeded".equals(phase)) {
        sleep 1
        phase = openshift.raw("get pod ${pod} -o custom-columns=POD:.status.phase --no-headers").out.trim()
        echo "phase = " + phase
    }
}

String newJob(String commandLine) {

    jobId = jobId()

    openshift.raw("create job ${jobId} --image=${IMAGE_NAME} -- ${commandLine}")

    return jobId
}

String params(Map options, List config) {

    params = " --insecure"
    config.each { item ->

        item.each { key, value ->

            option = options[key]

            if (option != null) {
                params += option
                if (option.endsWith("=")) {
                    params += "'${value}'"
                }
            }
        }
    }

    return params
}

String params(Map options, Map config) {

    params = " --insecure"
    config.each { key, value ->

        echo "${key} :: ${value}"

        option = options[key]

        if (option != null) {
            params += option
            if (option.endsWith("=")) {
                params += "'${value}'"
            }
        }
    }

    return params
}

pipeline {

    agent {
        label "master"
    }

    stages {

        stage("Initialize environment") {
            steps {
                script {

                    openshift.withCluster() {
                        openshift.withProject("${env.CICD_NAMESPACE}") {

                            json = openshift.raw("get configmap ${env.CONFIG_MAP} -o json").out
                            configMap = readJSON(text: json)

                            // globals
                            env.ACCESS_TOKEN = configMap.data.ACCESS_TOKEN
                            env.APP_BASE_NAME = configMap.data.APP_BASE_NAME
                            env.APP_OPEN_API_URL = configMap.data.APP_OPEN_API_URL
                            env.THREESCALE_SERVER_BASE_URL = configMap.data.THREESCALE_SERVER_BASE_URL
                            env.DESTINATION = "https://${env.ACCESS_TOKEN}@${env.THREESCALE_SERVER_BASE_URL}"

                            // backends
                            openapi_yaml = configMap.data.openapi_yaml
                            backends = readYaml(text: openapi_yaml)
                            BACKENDS = backends.backends

                            // applications
                            application_yaml = configMap.data.application_yaml
                            applications = readYaml(text: application_yaml)
                            APPLICATIONS = applications.applications

                            // application plans
                            applicationPlan_yaml = configMap.data.applicationPlan_yaml
                            applicationPlans = readYaml(text: applicationPlan_yaml)
                            APPLICATION_PLANS = applicationPlans.plans
                        }
                    }
                }
            }
        }

        stage("Import OpenAPI spec") {
            steps {
                script {

                    openshift.withCluster() {
                        openshift.withProject("${env.CICD_NAMESPACE}") {

                            def options = [
                                    'activeDocsHidden'         : ' --activedocs-hidden',
                                    'backendApiHostHeader'     : ' --backend-api-host-header=',
                                    'backendApiSecretToken'    : ' --backend-api-secret-token=',
                                    'defaultCredentialsUserKey': ' --default-credentials-userkey=',
                                    'output'                   : ' --output=',
                                    'oidcIssuerEndpoint'       : ' --oidc-issuer-endpoint=',
                                    'oidcIssuerType'           : ' --oidc-issuer-type=',
                                    'overridePrivateBaseUrl'   : ' --override-private-base-url=',
                                    'overridePrivateBasePath'  : ' --override-private-basepath=',
                                    'overridePublicBasePath'   : ' --override-public-basepath=',
                                    'prefixMatching'           : ' --prefix-matching',
                                    'productionPublicBaseUrl'  : ' --production-public-base-url=',
                                    'skipValidation'           : ' --skip-openapi-validation',
                                    'stagingPublicBaseUrl'     : ' --staging-public-base-url=',
                                    'targetSystemName'         : ' --target_system_name='
                            ]

                            BACKENDS.each { backend ->

                                echo "backend = ${backend}"
                                def params = params(options, backend)

                                commandLine = "3scale import openapi ${params} --destination=${env.DESTINATION} ${backend.openApiUrl}"
                                echo "commandLine = ${commandLine}"
                                jobId = newJob(commandLine)

                                timeout(time: 20, unit: 'SECONDS') {
                                    node {
                                        pod = waitForPod(jobId)

                                        waitForSuccessfulCompletion(pod)

                                        jobOutput = openshift.raw("logs ${pod}").out
                                        echo "jobOutput = " + groovy.json.JsonOutput.prettyPrint(jobOutput)
                                    }
                                }

                                openshift.raw("delete job ${jobId}")
                            }
                        }
                    }
                }
            }
        }

        stage('Get list of services') {
            steps {
                script {

                    openshift.withCluster() {
                        openshift.withProject("${env.CICD_NAMESPACE}") {

                            commandLine = "3scale service list --insecure --output=json ${env.DESTINATION}"
                            jobId = newJob(commandLine)

                            timeout(time: 20, unit: 'SECONDS') {
                                node {
                                    pod = waitForPod(jobId)

                                    waitForSuccessfulCompletion(pod)

                                    jobOutput = openshift.raw("logs ${pod}").out
                                    echo "jobOutput = " + groovy.json.JsonOutput.prettyPrint(jobOutput)
                                }
                            }

                            openshift.raw("delete job ${jobId}")
                        }
                    }
                }
            }
        }


        stage("Create Application Plans") {
            steps {
                script {

                    openshift.withCluster() {
                        openshift.withProject("${env.CICD_NAMESPACE}") {

                            def options = [
                                    'approvalRequired': ' --approval-required=',
                                    'costPerMonth'    : ' --cost-per-month=',
                                    'default'         : ' --default',
                                    'disabled'        : ' --disabled',
                                    'enabled'         : ' --enabled',
                                    'hide'            : ' --hide',
                                    'name'            : ' --name=',
                                    'output'          : ' --output=',
                                    'publish'         : ' --publish',
                                    'setupFee'        : ' --setup-fee=',
                                    'trialPeriodDays' : ' --trial-period-days='
                            ]

                            APPLICATION_PLANS.each { plan ->

                                echo "plan = ${plan}"
                                def params = params(options, plan)

                                commandLine = "3scale application-plan apply ${params} ${env.DESTINATION} ${plan.targetSystemName} ${plan.systemName}"
                                echo "commandLine = ${commandLine}"
                                jobId = newJob(commandLine)

                                timeout(time: 20, unit: 'SECONDS') {
                                    node {
                                        pod = waitForPod(jobId)

                                        waitForSuccessfulCompletion(pod)

                                        jobOutput = openshift.raw("logs ${pod}").out
                                        echo "jobOutput = ${jobOutput}"
                                    }
                                }

                                openshift.raw("delete job ${jobId}")
                            }
                        }
                    }
                }
            }
        }

        stage("Create Applications") {
            steps {
                script {

                    openshift.withCluster() {
                        openshift.withProject("${env.CICD_NAMESPACE}") {

                            def options = [
                                    'account'       : ' --account=',
                                    'applicationKey': ' --application-key=',
                                    'description'   : ' --description=',
                                    'name'          : ' --name=',
                                    'output'        : ' --output=',
                                    'plan'          : ' --plan=',
                                    'redirectUrl'   : ' --redirect-url=',
                                    'resume'        : ' --resume',
                                    'service'       : ' --service=',
                                    'suspend'       : ' --suspend',
                                    'userKey'       : ' --user-key='
                            ]

                            APPLICATIONS.each { application ->

                                echo "application = ${application}"
                                def params = params(options, application)

                                commandLine = "3scale application apply ${params} ${env.DESTINATION} ${application.systemName}"
                                echo "commandLine = ${commandLine}"
                                jobId = newJob(commandLine)

                                timeout(time: 20, unit: 'SECONDS') {
                                    node {
                                        pod = waitForPod(jobId)

                                        waitForSuccessfulCompletion(pod)

                                        jobOutput = openshift.raw("logs ${pod}").out
                                        echo "jobOutput = ${jobOutput}"
                                    }
                                }

                                openshift.raw("delete job ${jobId}")
                            }
                        }
                    }
                }
            }
        }

    }
}
