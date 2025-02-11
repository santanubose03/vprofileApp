def COLOR_MAP = [
  'SUCCESS' : 'good',
  'FAILURE' : 'danger',
  'ABORT' : 'warning'
]

def dockerImage

pipeline{
    agent any
    tools {
        maven "MAVEN3.9.9"
        jdk "JDK17"
    }

    environment{
        registryCredentials = "ecr:us-east-1:AWS_CREDS" 
        vprofileRegistry = "https://180294204145.dkr.ecr.us-east-1.amazonaws.com"
        imageName = "180294204145.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg"
        clusterName = 'vprofile'
        serviceName = 'vprofileappsrv'
    }

   
    stages{
        stage('git checkout'){
            steps {
                 git branch: 'docker', url: 'https://github.com/santanubose03/vprofile-project-hkhcoder.git'
                
            }
        }

        stage('Build and Archive'){
          steps {
                sh 'mvn install -DskipTests'
            }
          post {
             always {
               echo 'Now archiving' 
                   script {
                      def warFile = sh(script: "ls target/*.war | head -n 1", returnStdout: true).trim()
                    
                      if (fileExists(warFile)) {
                        archiveArtifacts artifacts: warFile
                       }

                      else {
                        echo "No WAR file found. Skipping artifact archiving."
                       }
                    }
                }
            }    
        }

        stage('Unit Test'){
           steps{
              sh 'mvn test'
            }
        }

        // stage('Integration Test'){
        //   steps{
        //       sh 'mvn verify -DskipUnitTests'
        //   }
        // }

        stage('Code Analysis with Checkstyle'){
          steps{
              sh 'mvn checkstyle:checkstyle'
           }
          post{
               success{
                     echo 'Generated Analysis Result'
                }
            }
        }

        stage('Sonarqube Analysis'){
          
           environment{
             scannerHome = tool 'SONAR6.2'  
            }

           steps{
               withSonarQubeEnv('sonarserver') {
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }      
        } 

        stage('Quality Gate'){
            steps{
                timeout(time: 10, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }

        }

        stage('Publish Artifact'){
           steps{
               nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "54.234.122.250:8081",
                    groupId: 'vprofile',
                    version: "${env.BUILD_NUMBER}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-app-maven',
                    credentialsId: 'NEXUS_LOGIN',
                    artifacts: [
                        [artifactId: 'vproapp',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                 )
           }
        } 

        stage('Build Docker Image'){
            steps{
                script{
                    // dockerImage = docker.build(vprofileImage + "${env.BUILD_NUMBER}", "./Docker-files/app/multistage/")
                     dockerImage = docker.build(imageName + ":${SEMANTIC_VERSION}.${env.BUILD_NUMBER}", "./Docker-files/app/multistage/")
                }
            }
        }

        stage('Upload Docker Image'){
            steps{
                script{
                    docker.withRegistry(vprofileRegistry,registryCredentials) {
                       echo "Image Name: ${imageName}"
                       echo "Semantic Version: ${SEMANTIC_VERSION}"
                       echo "Build Number: ${env.BUILD_NUMBER}"
                       // dockerImage.push("${env.BUILD_NUMBER}")
                       dockerImage.push("v${SEMANTIC_VERSION}.${env.BUILD_NUMBER}")
                       dockerImage.push('latest')
                    }
                     sh 'docker rmi -f $(docker images -a -q) || true'
                }
            }
        }

        stage('deploying containers from ECR'){
            steps{
                withAWS(credentials: 'AWS_CREDS' , region:'us-east-1') {
                  
                  sh "aws ecs update-service --cluster ${clusterName} --service ${serviceName} --force-new-deployment"

                 }
            }
        }

        
    }

    post{
        always{
            slackSend channel:'#all-freelencer',
                          color: COLOR_MAP[currentBuild.currentResult],
                          message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} build_url:${BUILD_URL}"
        }
    }
    
}
