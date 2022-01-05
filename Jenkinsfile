/* Change the Jenkins dashboard display name*/
currentBuild.displayName = "Sonarqube-Mule-demo-#"+ currentBuild.number

pipeline {
    agent any
    
    environment {
        ANYPOINT_CREDS = credentials('ANYPOINT_CREDS')
        ANYPOINT_PLATFORM_CREDS = credentials('ANYPOINT_PLATFORM_CREDS')
        BUILD_VERSION = "${currentBuild.number}.RELEASE"
        APP_NAME="sonar-demo-anup"
        MULE_VERSION="4.4.0"
        KEY_INT = 'IF Requireds' 
        MULE_JAR_NAME='sonar-demo-1.0.0-SNAPSHOT-mule-application.jar'
	    API_ID_INT = '00000'
    }
    
    tools {
        maven 'M3'
    }

    stages {

        /* Checkout Code From SCM
        Adjust in credentials for private repo with appropriate syntax */

        stage('Checkout code from git repository') {
            steps{
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/asalokhe/mule-jenkins-sonarqube.git']]])
            }
        }

        /* Run Munit test before quality check as we can add the Code Coverage Quality Gate validation in Sonar*/

        stage('Run Mule munit test') {
            steps {
            configFileProvider([configFile(fileId: '5d13b8ee-42ad-4a58-afce-79dbdaa62a2a', variable: 'MAVEN_SETTINGS')]) {
            sh "printenv | sort"
            sh 'mvn -B -s "$MAVEN_SETTINGS" clean test'
                }
            }
        }
        
        /* Pre-Req: 
            1. Plugin in Jenkins: "SonarQube Scanner for Jenkins" 
            2. Jenkins: Configure to the SonarQube Server instance in the "Configure System" settings in Jenkins "http://sonarqube:9000" (No trailing slash)
            3. SonarQube: Remove ".xml" from XML Language.
            4. Run the command to do the code qualuty analysis 
              mvn sonar:sonar -Dsonar.host.url=http://mysonarqubeserver:9000 -Dsonar.sources=src/ 
              sonar.host.url is not always required as withSonarQubeEnv('SonarQube') is configured in Jenkins and in local settings.xml has 
              profile section. */

        stage("Perform quality check in SonarQube") {
           steps{
               withSonarQubeEnv('SonarQube') {
                   sh 'mvn sonar:sonar -Dsonar.sources=src/main/'
               }
           }
       }
       
       /* Pre-Req: 
        1. Plugin in Jenkins: "SonarQube Scanner for Jenkins" 
        2. SonarQube: Quality gate needs to be predefined in sonarqube.
        3. SonarQube: http://jenkins:8080/sonarqube-webhook/ needs to be predefined in sonar-server.
        Based on the analysis, Webhook url will be called. This step checks the status & decide whether to proceed or fail the further pipeline */

        stage("Perform qualitygate check") {
           steps{
               waitForQualityGate abortPipeline: true
           }
       }
        
        /* Unit Test the Code and Build the *.Jar file
           NOTE: -DskipTests to skip Munitunit testing as Munit testing is executed during quality check */

        stage('Package the code') {
            steps {
            configFileProvider([configFile(fileId: '5d13b8ee-42ad-4a58-afce-79dbdaa62a2a', variable: 'MAVEN_SETTINGS')]) {
            sh "printenv | sort"
            sh 'mvn -B -s "$MAVEN_SETTINGS" clean package -DskipTests'
                }
            }
        }
        
        /* Deploy the Sandbox to Sandbox
           Use mule:deploy to instruct the plugin to deploy alredy built jar - Provide the mule.artifact path
           Use mvn deploy if you want to rebuild the jar and deploy. 
           Reference: https://docs.mulesoft.com/mule-runtime/4.4/mmp-concept
           NOTE: -DskipTests to skip Munitunit testing.   */
           
        stage('Deploy packaged code to cloudhub Sandbox') {
            steps {
            configFileProvider([configFile(fileId: '5d13b8ee-42ad-4a58-afce-79dbdaa62a2a', variable: 'MAVEN_SETTINGS')]) {
            sh "printenv | sort"
            sh 'mvn -B -s "$MAVEN_SETTINGS" mule:deploy \
                    -Dmule.artifact=/var/jenkins_home/workspace/demo-1/target/$MULE_JAR_NAME \
                    -Danypoint.account.username=$ANYPOINT_CREDS_USR \
                    -Danypoint.account.password=$ANYPOINT_CREDS_PSW \
                    -Danypoint.runtime.environment=Sandbox \
                    -Danypoint.platform.client_id=$ANYPOINT_PLATFORM_CREDS_USR \
                    -Danypoint.platform.client_secret=$ANYPOINT_PLATFORM_CREDS_PSW \
                    -Dmule.version=$MULE_VERSION \
                    -Ddeploy.applicationName=$APP_NAME \
                    -Dhttp.host=0.0.0.0 \
                    -Dhttp.port=8081'
                }
            }
        }
    }
}