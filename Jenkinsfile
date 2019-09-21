import hudson.model.*
import groovy.json.JsonSlurper
def BuildVersion
def Current_version
def NextVersion
def Commit_Id
import hudson.FilePath
import hudson.model.Node
import hudson.model.Slave
import jenkins.model.Jenkins
import groovy.time.*
 pipeline {

     options {
         timeout(time: 30, unit: 'MINUTES')
     }
    environment {
    registry = "rozdockerforever/dev"
    registryCredential = 'dockerhub'
    }
     agent { label 'slave' }
     stages {
         stage('Checkout') {
             steps {
                 script {
                     node('master'){
                         dir('Release') {
                             deleteDir()
                             checkout([$class: 'GitSCM', branches: [[name: 'Prod']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github-roz', url: "https://github.com/roz-Devops/Release.git"]]])
                             path_json_file = sh(script: "pwd", returnStdout: true).trim() + '/' + 'Prod' + '.json'
                             Current_version = Return_Json_From_File("$path_json_file").release.services.intapi.version
                             echo("Current_version Is in master: ${Current_version}")
                         }
                     }
                     
                     dir('INT_DB') {
                         deleteDir()
                         checkout([$class: 'GitSCM', branches: [[name: 'Dev']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github-roz', url: "https://github.com/roz-Devops/INT_DB.git"]]])
                         Commit_Id = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                         BuildVersion = Current_version + '_' + Commit_Id
                         last_digit_current_version = sh(script: "echo $Current_version | cut -d'.' -f3", returnStdout: true).trim()
                         NextVersion = sh(script: "echo $Current_version | cut -d. -f1", returnStdout: true).trim() + '.' + sh(script: "echo $Current_version |cut -d'.' -f2", returnStdout: true).trim() + '.' + (Integer.parseInt(last_digit_current_version) + 1)
                         println("Checking the build version: $BuildVersion")

                     }
                 }
             }
         }
         stage('UT') {
             steps {
                 println('Will be added soon ') 

             }
         }
         stage('Build') {
             steps {
                 script {
                     dir('INT_DB') {
                         try {
                           docker.build("db:$BuildVersion")
                           println("Docker image is successfully built")  

                         }
                         catch (exception) {
                             println "Docker image build failed"
                             currentBuild.result = 'FAILURE'
              
                         }

                     }
                     
                 }


             }
         }
        stage('Test the container is runnable') {
            steps {
                script {
                     sh 'ls'
                     sh 'pwd'  
                     sh '${Commit_Id}'
                         sh ''' $(cat README.md | head -1):${Commit_Id} && if [ ?$ -eq 0 ]; then echo 'Launch SUCCESS' && docker stop mongodb; else exit 1; fi'''
                       
                     }
                   }
         }
         stage('Push image to repository'){
             steps{
                 script{
                     try{
                         withCredentials([usernamePassword(credentialsId: 'docker-cred-id', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                                sh "docker login -u=${DOCKER_USERNAME} -p=${DOCKER_PASSWORD}"
                                sh "docker tag db:$BuildVersion $registry:db_$BuildVersion"
                                sh "docker push $registry:db_$BuildVersion"
                                
                         }
                         }
                     catch (exception){
                         println "The image pushing to dockehub failed"
                         currentBuild.result = 'FAILURE'
                     }
                 }
             }
         }
         stage('Triggering E2E-CI job'){
            
             steps{
                script {
                    node('master') {
                    build job: 'E2E-CI', parameters: [string(name: 'triggered_by', value: 'intdb'), string(name: 'next_version', value: NextVersion), string(name: 'Image_version', value: 'db_' + BuildVersion)]
                }

                    }


                }
            }

        }

     }
def Return_Json_From_File(file_name){
    return new JsonSlurper().parse(new File(file_name))
}
