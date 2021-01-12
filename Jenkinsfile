
#!/usr/bin/env groovy
@Library('pipeline-library')

def services = [:]
def credentials = [:]
def ocp_clusters = [:]

pipeline {
    agent none
    environment {
        // Global variables:
        OCP_TOKEN = ''
        ARTIFACT_URL = ''
        // Git Credential & This Git Repo URL:
        GIT_CREDENTIAL = 'my-ssh-key-git'   // Deployment key setup in github's repo with private key saved in Cloudbees CI with this name.
        GIT_REPO_URL = 'git@github.domain.com:organization/repo.git'
    }
    options {
        // only keeps last 30 builds and artifacts
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '30'))
        disableConcurrentBuilds()
    }
    parameters {
        choice(name: 'ENV_TO_DEPLOY', choices: ['DEV','UAT','PRE','PRO'], description: 'Choose between deploying to DEV, UAT, PRE or PRO')
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2', 'app3', 'app4', 'app5', 'app6'], description: 'Choose app to deploy')
    }
    stages {
        stage('Clone Repository') {
            steps {
                checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: 'refs/remotes/origin/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout'], [$class: 'CheckoutOption', timeout: 1], [$class: 'CloneOption', honorRefspec: true, noTags: true, reference: '', shallow: true, timeout: 1]], submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${GIT_CREDENTIAL}", url: "${GIT_REPO_URL}"]]]
            }
        }
        stage("read yaml") {
            //agent any
            steps {
                script {
                    services = readYaml file: 'services.yml'
                    credentials = readYaml file: 'credentials.yml'
                    ocp_clusters = readYaml file: 'ocp_clusters.yml'
                    approvers = readYaml file: 'approvers.yml'
                }
            }
        }
        stage("Check params") {
            agent any
            steps {
                script{
                    echo "APP_TO_DEPLOY: ${params.APP_TO_DEPLOY}"
                    echo services."${params.APP_TO_DEPLOY}".nexus_group_id
                    echo services."${params.APP_TO_DEPLOY}".nexus_version.toString()
                    echo services."${params.APP_TO_DEPLOY}".onehr_springcloud_conf_properties_version
                    echo services."${params.APP_TO_DEPLOY}".nexus_artifact_id
                    echo services."${params.APP_TO_DEPLOY}".nexus_packaging

                    // https://support.cloudbees.com/hc/en-us/articles/204986450-Pipeline-How-to-manage-user-inputs
                    def userInput = input(
                    id: 'userInput', message: 'Let\'s promote?', parameters: [
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_group_id , description: 'nexus group id', name: 'NEXUS_GROUP_ID'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_version_dev , description: 'nexus version dev', name: 'NEXUS_VERSION_DEV'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_version_uat , description: 'nexus version uat', name: 'NEXUS_VERSION_UAT'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_version_pre , description: 'nexus version pre', name: 'NEXUS_VERSION_PRE'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_version_pro , description: 'nexus version pro', name: 'NEXUS_VERSION_PRO'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_artifact_id , description: 'nexus artifact id', name: 'NEXUS_ARTIFACT_ID'],
                    [$class: 'TextParameterDefinition', defaultValue: services."${params.APP_TO_DEPLOY}".nexus_packaging , description: 'nexus packaging', name: 'NEXUS_PACKAGING']
                    ])
                    echo ("NEXUS GROUP ID: "+userInput['NEXUS_GROUP_ID'])
                    switch("${params.ENV_TO_DEPLOY}") {
                        case 'dev':
                            echo ("Nexus Version DEV: "+userInput['NEXUS_VERSION_DEV'])
                            break;
                        case 'uat':
                            echo ("Nexus Version UAT: "+userInput['NEXUS_VERSION_UAT'])
                            break;
                        case 'pre':
                            echo ("Nexus Version PRE: "+userInput['NEXUS_VERSION_PRE'])
                            break;
                        case 'pro':
                            echo ("Nexus Version PRO: "+userInput['NEXUS_VERSION_PRO'])
                            break;
                        default:
                            echo ("Nexus Version PRO: "+userInput['NEXUS_VERSION_PRO'])
                            break;
                    }
                    echo ("Nexus Artifact ID: "+userInput['NEXUS_ARTIFACT_ID'])
                    echo ("Nexus Packaging Type: "+userInput['NEXUS_PACKAGING'])

                    services."${params.APP_TO_DEPLOY}".nexus_group_id = userInput['NEXUS_GROUP_ID']
                    services."${params.APP_TO_DEPLOY}".nexus_version_dev = userInput['NEXUS_VERSION_DEV']
                    services."${params.APP_TO_DEPLOY}".nexus_version_uat = userInput['NEXUS_VERSION_UAT']
                    services."${params.APP_TO_DEPLOY}".nexus_version_pre = userInput['NEXUS_VERSION_PRE']
                    services."${params.APP_TO_DEPLOY}".nexus_version_pro = userInput['NEXUS_VERSION_PRO']
                    services."${params.APP_TO_DEPLOY}".nexus_artifact_id = userInput['NEXUS_ARTIFACT_ID']
                    services."${params.APP_TO_DEPLOY}".nexus_packaging = userInput['NEXUS_PACKAGING']
                }
            }
        }
        stage('Approval') {
            // no agent, so executors are not used up when waiting for approvals
            agent none
            steps {
                script {
                    def APPROVERS
                    switch("${params.ENV_TO_DEPLOY}") {
                        case 'dev':
                            APPROVERS = approvers.APPROVERS_DEV
                            break;
                        case 'uat':
                            APPROVERS = approvers.APPROVERS_UAT
                            break;
                        case 'pre':
                            APPROVERS = approvers.APPROVERS_PRE
                            break;
                        case 'pro':
                            APPROVERS = approvers.APPROVERS_PRO
                            break;
                        default:
                            APPROVERS = approvers.APPROVERS_PRE
                            break;
                    }

                    def deploymentDelay = input id: 'Deploy', message: "Deploy to ${params.ENV_TO_DEPLOY}?", submitter: "${APPROVERS}", parameters: [choice(choices: ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11', '12', '13', '14', '15', '16', '17', '18', '19', '20', '21', '22', '23', '24'], description: 'Hours to delay deployment?', name: 'deploymentDelay')]
                    sleep time: deploymentDelay.toInteger(), unit: 'HOURS'
                }
            }
        }
        stage("Resolving artifact from Nexus") {
            agent none
            steps {
                script {
                    def nexus_version = ''
                    switch("${params.ENV_TO_DEPLOY}") {
                        case 'dev':
                            nexus_version = services."${params.APP_TO_DEPLOY}".nexus_version_dev
                            break;
                        case 'uat':
                            nexus_version = services."${params.APP_TO_DEPLOY}".nexus_version_uat
                            break;
                        case 'pre':
                            nexus_version = services."${params.APP_TO_DEPLOY}".nexus_version_pre
                            break;
                        case 'pro':
                            nexus_version = services."${params.APP_TO_DEPLOY}".nexus_version_pro
                            break;
                        default:
                            nexus_version = services."${params.APP_TO_DEPLOY}".nexus_version_pro
                            break;
                    }
                    ARTIFACT_URL = mavenNexusResolver(
                            groupId: services."${params.APP_TO_DEPLOY}".nexus_group_id,
                            artifactId: services."${params.APP_TO_DEPLOY}".nexus_artifact_id,
                            version: "${nexus_version}",
                            packaging: services."${params.APP_TO_DEPLOY}".nexus_packaging)
                    currentBuild.displayName = "#$env.BUILD_ID "+services."${params.APP_TO_DEPLOY}".nexus_artifact_id+": ${nexus_version}"
                }
                echo "Artifact to deploy: ${ARTIFACT_URL}"
            }
        }
        stage ("Setting up OCP Credentials") {
            steps {
                script {
                    echo "credentials: $credentials"
                    credentials.each { credkey, credvalue ->
                        if ( "$credkey" == "${params.ENV_TO_DEPLOY}" ) {
                            echo "okay"
                            credvalue.each { cluster ->
                                echo "cluster key: $cluster.key"
                                echo "cluster value: $cluster.value"
                                echo "app to deploy: "+"${params.APP_TO_DEPLOY}"
                                echo "scope: "+services."${params.APP_TO_DEPLOY}".scope
                                switch(services."${params.APP_TO_DEPLOY}".scope) {
                                    case 'local':
                                        OCP_TOKEN = cluster.value.ocp_token_local
                                        break;
                                    case 'global':  //global apps start with "global" in their names
                                        OCP_TOKEN = cluster.value.ocp_token_global
                                        break;
                                    default:
                                        OCP_TOKEN = cluster.value.ocp_token_local
                                        break;
                                }
                            }
                        }
                    }
                    echo "OCP_TOKEN: ${OCP_TOKEN}"
                }
            }
        }
        stage('Deploy') {
            agent {
                //using ocp-deploy slave to execute this stage
                label 'ocp-deploy'
            }
            steps {
                script {
                        echo "Deployment of ${params.APP_TO_DEPLOY} in Project/Namespace: "+services."${params.APP_TO_DEPLOY}"."ocp_namespace_${params.ENV_TO_DEPLOY}"
                        credentials."${params.ENV_TO_DEPLOY}".each { cluster_to_deploy ->
                            echo "Environment: ${params.ENV_TO_DEPLOY}"
                            echo "Application: ${params.APP_TO_DEPLOY}"
                            echo "OCP Cluster: $cluster_to_deploy.key ("+ocp_clusters.get(cluster_to_deploy.key)+")"
                            echo "Openshift Namespace/Project: "+services."${params.APP_TO_DEPLOY}"."ocp_namespace_${params.ENV_TO_DEPLOY}"

                            openshiftDeploy(
                                ocpCluster: ocp_clusters.get(cluster_to_deploy.key),         // OpenShift region/cluster to deploy
                                ocpProject: services."${params.APP_TO_DEPLOY}"."ocp_namespace_${params.ENV_TO_DEPLOY}" ,       // OpenShift project/namespace to deploy
                                ocpApplication: "${params.APP_TO_DEPLOY}",  // OpenShift application name
                                ocpTemplate: 'bwce',                   // Bussiness Works Container Edition OpenShift template
                                ocpTokenCredentialId: "${OCP_TOKEN}", // OpenShift deploy credential id in Jenkins
                                ocpTemplateParams: [ // bwe template params 
                                    ARTIFACT_URL: "${ARTIFACT_URL}",    // EAR URL in nexus
                                    BW_PROFILE: 'Docker',                 // BW profile
                                    APP_CONFIG_PROFILE: "${params.ENV_TO_DEPLOY}",            // Profile in the configuration Service
                                    //APP_CONFIG_PROFILE: 'PRE',            // Profile in the configuration Service
                                    SPRING_CLOUD_CONFIG_SERVER_URL: 'http://config-service:8080' // Spring Cloud configuration service URL
                                ]
                            )
                        }
                }
            }
        }
    }
}