def policyEvaluation = null

pipeline {
    agent any

    stages {
        stage('Preparation') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            tools {
                jdk '1.8'
                maven 'Maven'
            }
            steps {
                sh "mvn clean package -DskipTests"
            }
        }
        stage('IQ Policy Check') {
            steps {
                script {
                    // env.GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1')
                    try {
                        policyEvaluation = nexusPolicyEvaluation(failBuildOnNetworkError: false, iqScanPatterns: [[scanPattern: 'target/struts2-rest-showcase.war']], 
                        reachability: [
                            javaAnalysis: [
                                enable: true,
                                entrypointStrategy: 'ACCESSIBLE_CONCRETE',
                                namespaces: [
                                    [namespace: 'org.demo.rest.example']
                                ]
                            ]
                        ],
                        iqApplication: "struts2-rce-github", iqStage: 'build', jobCredentialsId: '')
                    } catch (err) {
                        echo "IQ Policy evaluation failed: ${err.message}"
                        policyEvaluation = err.policyEvaluation
                        // Note: To access err.policyEvaluation, approve in Jenkins → In-process Script Approval
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                // Create TAG
                def iqScanUrl = policyEvaluation?.applicationCompositionReportUrl ?: 'N/A'
                createTag nexusInstanceId: 'Nexus', tagAttributesJson: """{
                    \"IQScan\": \"${iqScanUrl}\",
                    \"JenkinsBuild\": \"${BUILD_URL}\"
                }""", tagName: "IQ-Policy-Evaluation_${currentBuild.number}"
                // Publish
                nexusPublisher nexusInstanceId: 'Nexus', nexusRepositoryId: 'maven-uat-hosted', packages: [[
                    $class: 'MavenPackage',
                    mavenAssetList: [[classifier: '', extension: '', filePath: './target/struts2-rest-showcase.war']],
                    mavenCoordinate: [artifactId: 'struts2-rest-showcase', groupId: 'org.apache.struts', packaging: 'war', version: '2.5.10']
                ]], tagName: "IQ-Policy-Evaluation_${currentBuild.number}"
                // Staging and Promote
                moveComponents destination: 'maven-prod-hosted', nexusInstanceId: 'Nexus', tagName: "IQ-Policy-Evaluation_${currentBuild.number}"
            
            }
        }
    }
}

