#!groovy

@Library('cib-pipeline-library') _

import de.cib.pipeline.library.Constants
import de.cib.pipeline.library.kubernetes.BuildPodCreator
import de.cib.pipeline.library.logging.Logger
import de.cib.pipeline.library.ConstantsInternal
import de.cib.pipeline.library.MavenProjectInformation
import groovy.transform.Field

@Field Logger log = new Logger(this)
@Field MavenProjectInformation mavenProjectInformation = null
@Field Map pipelineParams = [
    pom: ConstantsInternal.DEFAULT_MAVEN_POM_PATH,
    mvnContainerName: Constants.MAVEN_JDK_17_CONTAINER,
    uiParamPresets: [:],
    testMode: false
]

pipeline {
    agent {
        kubernetes {
            yaml BuildPodCreator.cibStandardPod()
                    .withContainerFromName(pipelineParams.mvnContainerName)
                    .asYaml()
            defaultContainer pipelineParams.mvnContainerName
        }
    }

    // Parameter that can be changed in the Jenkins UI
    parameters {
        booleanParam(
            name: 'INSTALL',
            defaultValue: true,
            description: 'Build and test'
        )
        booleanParam(
            name: 'DEPLOY',
            defaultValue: false,
            description: 'Deploy artifacts to artifacts.cibseven.org'
        )
        password(
            name: 'DEPLOY_MAVEN_CENTRAL_PASSWORD',
            defaultValue: '',
            description: 'Enter a password for deployment to Maven Central (no need to activate DEPLOY parameter above if you want just to deploy to Maven Central). SNAPSHOT version will not be deployed into Maven Central. If you will not change this value - you will not run deploy to Maven Central.'
        )
        string(
            name: 'COMMUNITY_USERNAME',
            defaultValue: '',
            description: 'Community username for publishing to Maven Central'
        )
        password(
            name: 'COMMUNITY_PASSWORD',
            defaultValue: '',
            description: 'Community password for publishing to Maven Central'
        )
        file(
            name: 'GPG_KEY_FILE',
            description: 'GPG private key file for signing Maven Central deployment'
        )
        password(
            name: 'GPG_KEY_TRUST',
            defaultValue: '',
            description: 'GPG key trust'
        )
    }

    options {
        buildDiscarder(
            logRotator(
                // number of build logs to keep
                numToKeepStr:'5',
                // history to keep in days
                daysToKeepStr: '15',
                // artifacts are kept for days
                artifactDaysToKeepStr: '15',
                // number of builds have their artifacts kept
                artifactNumToKeepStr: '5'
            )
        )
        // Stop build after 240 minutes
        timeout(time: 240, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Print Settings & Checkout') {
            steps {
                script {
                    printSettings()

                    def pom = readMavenPom file: pipelineParams.pom

                    // for overlays often no groupId is set as the parent groupId is used
                    def groupId = pom.groupId
                    if (groupId == null) {
                        groupId = pom.parent.groupId
                        log.info "parent groupId is used"
                    }

                    mavenProjectInformation = new MavenProjectInformation(groupId, pom.artifactId, pom.version, pom.name, pom.description)

                    log.info "Build Project: ${mavenProjectInformation.groupId}:${mavenProjectInformation.artifactId}, ${mavenProjectInformation.name} with version ${mavenProjectInformation.version}"

                    // Avoid Git "dubious ownership" error in checked out repository. Needed in
                    // build containers with newer Git versions. Originates from Jenkins running
                    // pipeline as root but repository being owned by user 1000. For more, see
                    // https://stackoverflow.com/questions/72978485/git-submodule-update-failed-with-fatal-detected-dubious-ownership-in-repositor
                    sh "git config --global --add safe.directory \$(pwd)"
                }
            }
        }

        stage('Maven install') {
            when {
                expression { params.INSTALL == true }
            }
            steps {
                script {
                    withMaven(options: [junitPublisher(disabled: false), jacocoPublisher(disabled: false)]) {
                        sh "mvn -T4 -Dbuild.number=${BUILD_NUMBER} install"
                    }

                    junit allowEmptyResults: true, testResults: ConstantsInternal.MAVEN_TEST_RESULTS
                }
            }
        }

        stage('Deploy to artifacts.cibseven.org') {
            when {
                expression { params.DEPLOY == true }
            }
            steps {
                script {
                    withMaven(options: []) {
                        sh "mvn -T4 -U -DskipTests clean deploy"
                    }
                }
            }
        }
        
        stage('Deploy to Maven Central') {
            when {
                allOf {
                    expression { params.DEPLOY_MAVEN_CENTRAL_PASSWORD.toString().isEmpty() == false } // "" is default value
                    expression { mavenProjectInformation.version.endsWith("-SNAPSHOT") == false }
                }
            }
            steps {
                script {
                   
                    // Check if GPG key content parameter exists
                    // if (!params.GPG_KEY_FILE) {
                    //    error "GPG key file parameter is required for Maven Central deployment"
                    // }

                    sh "echo Before"
                    sh "echo ${params.GPG_KEY_FILE}"
                    sh "echo After"

                    // Safely handle the file parameter
                    def fileContent = null
                    
                    // Check if we're running on master/controller node
                    node('built-in') {
                        // Read the file content using workspace-relative path
                        fileContent = readFile(file: "${env.WORKSPACE}/${params.GPG_KEY_FILE}")
                    }

                    // Write the key to a temporary file
                    def keyFile = "./gpg-key.asc"

                    if (fileContent != null) {
                        writeFile file: keyFile, text: fileContent

                        // Print confirmation
                        echo "File has been written to ${keyFile}"
                        
                        // Optionally verify the file contents
                        sh "cat ${keyFile}"
                        
                    } else {
                        error "Failed to read key file content"
                    }
                    
                    
                    
    
                    // Import the key
                    sh """
                        cat ${keyFile}
                        gpg --batch --import ${keyFile}
                        rm -f ${keyFile}
                        gpg --list-keys
                    """

                    // Write the trust to a temporary file
                    def gpgKeyTrust = params.GPG_KEY_TRUST.toString()
                    def trustFile = "./gpg-key.trust"
                    writeFile file: trustFile, text: gpgKeyTrust
                    
                    // Import the key trust
                    sh """
                        echo ${gpgKeyTrust}
                        gpg --batch --import-ownertrust ${trustFile}
                        cat ${trustFile}
                        rm -f ${trustFile}
                        gpg --list-keys
                    """
                    
                    withMaven(options: []) {
                        sh """
                            export 
                            mvn -T4 -U \
                                -Dcommunity.username="${params.COMMUNITY_USERNAME}" \
                                -Dcommunity.password="${params.COMMUNITY_PASSWORD}" \
                                --global-settings settings.xml \
                                clean deploy \
                                -Psonatype-oss-release \
                                -Dskip.cibseven.release=true \
                                -DskipTests \
                                -Dgpg.passphrase="${params.DEPLOY_MAVEN_CENTRAL_PASSWORD}" \
                                -Dgpg.keyname=\$(gpg --list-keys --with-colons | grep pub | cut -d: -f5)
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                log.info 'End of the build'
            }
        }

        success {
            script {
                log.info '✅ Build successful'
                if (params.RELEASE_BUILD == true) {
                    notifyResult(
                        office365WebhookId: pipelineParams.office365WebhookId,
                        message: "Application was successfully released with version ${mavenProjectInformation.version}"
                    )
                }
            }
        }

        unstable {
            script {
                log.warning '⚠️ Build unstable'
            }
        }

        failure {
            script {
                log.warning '❌ Build failed'
                if (env.BRANCH_NAME == 'master') {
                    notifyResult(
                        office365WebhookId: pipelineParams.office365WebhookId,
                        message: "Access build info at ${env.BUILD_URL}"
                    )
                }
            }
        }

        fixed {
            script {
                log.info '✅ Previous issues fixed'
                if (env.BRANCH_NAME == 'master') {
                    notifyResult(
                        office365WebhookId: pipelineParams.office365WebhookId,
                        message: "Access build info at ${env.BUILD_URL}"
                    )
                }
            }
        }
    }
}
