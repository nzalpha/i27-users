pipeline {
    agent {
        label 'K8-slave'
    }
    tools {
        maven 'mvn 3.8'
        jdk 'jdk-17'
    }
    environment {
        APPLICATION_NAME = "user"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB= "docker.io/nawaz004"
        DOCKER_CREDS= credentials('docker_cred')
    }

    parameters {
        choice (name: 'buildOnly', choices: 'no\nyes', description: "Only Build")
        choice (name: 'ScandOnly', choices: 'no\nyes', description: "Only Scan")
        choice (name: 'DockerPushOnly', choices: 'no\nyes', description: "Only Push to registry")
        choice (name: 'DeploytoDev', choices: 'no\nyes', description: "Only deploy to dev")
        choice (name: 'DeploytoTest', choices: 'no\nyes', description: "Only deploy to test")
        choice (name: 'DeploytoStage', choices: 'no\nyes', description: "Only deploy to stage")
        choice (name: 'DeploytoProd', choices: 'no\nyes', description: "Only deploy to prod")

    }


    stages {
        stage ('Build') {
            when{
                anyOf{
                    expression{
                        params.buildOnly == 'yes'
                    }
                }
            }


            // This step will take care of building the application
            steps {
                script{
                 buildApp().call()
                }
            }
        }
      
    

        // stage('Testing')
        // {
        //     steps{
        //         echo "Executing unit test for ${env.APPLICATION_NAME} "
        //         sh 'mvn test'
        //     }

        //     post {
        //         always {
        //             junit 'target/surefire-reports/*.xml'
        //             jacoco execPattern: 'target/jacoco.exec'
        //         }
        //     }
        // }


        stage("Docker build and Push")
        {

            when{
                anyOf{
                    expression{
                       
                        params.DockerPushOnly == 'yes'
                    }
                }
            }
            steps{
                script{
                    dockerBuildAndPush().call()
                }
                
            }
        }


        stage("Deploy to Dev")
        {
            when{
                anyOf{
                    expression{
                        params.DeploytoDev == 'yes'
                    }
                }
            }
            steps{
                script{
                imageValidation().call()    
                dockerDeploy('dev','5232','8232').call()
                }
            }
        }

           stage("Deploy to Test")
        {
            when{
                anyOf{
                    expression{
                        params.DeploytoTest == 'yes'
                    }
                }
            }
            steps{
                script{
                    imageValidation().call()   
              dockerDeploy('test','6232','8232').call()
                }
            }
        }

           stage("Deploy to Stage")
            {
            when{
                anyOf{
                    expression{
                        params.DeploytoStage == 'yes'
                    }
                }
            }
            steps{
                script{
                    imageValidation().call()   
              dockerDeploy('stage','7232','8232').call()
                }
            }
             }


           stage("Deploy to Prod")
            {
            when{
                allOf{
                anyOf{
                    expression{
                        params.DeploytoProd == 'yes'
                    }
                }
                anyOf {
                branch 'release/*'
                 }

                }
            }
            steps{
                 timeout(time: 300, unit: 'SECONDS') {
                    input message: "Deploying to ${env.APPLICATION_NAME} to production ???", ok: 'yes', submitter: 'ta'
                }
                script{
                    imageValidation().call()   
              dockerDeploy('prod','8232','8232').call()
                }
            }
             }

    }

}



// Method to reuse the deployment code for different environment

def dockerDeploy(envDeploy,hostPort, containerPort){
    return{
         echo ("-----------Deploying to ${envDeploy} Env---------")
                withCredentials([usernamePassword(credentialsId: 'ali_docker_vm_cred', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    script{
                         sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                    
                    try{
                        //Stop the container
                      sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop  ${env.APPLICATION_NAME}-${envDeploy}"
                        // Remove the container
                      sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm  ${env.APPLICATION_NAME}-${envDeploy}"   
                    }  catch(err)
                    {
                        echo "Error Caught: $err"
                       
                    }
                     echo " Creating Container"
                       sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p ${hostPort}:${containerPort} --name ${env.APPLICATION_NAME}-${envDeploy} ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                    }
                }
    }
}

def imageValidation(){
    return{
        println ("Pulling the docker image")
        try{
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
        catch(Exception e){
            println ("Docker image with this tag doesnt exist, so creating the image")
            buildApp().call()
            dockerBuildAndPush().call()
        }
    }
}

def buildApp(){
    return{
         echo "Building the  Application"
                //mvn command 
                sh 'mvn clean package -DskipTests=true'
                archiveArtifacts artifacts: 'target/*.jar'
    }
}

def dockerBuildAndPush(){
    return{
        sh "cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd/"
        echo "------------Building Docker Image--------"
        sh "docker build --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd/"
        echo "Build Done"
        echo "-----------Docker Login--------------"
        sh "docker login -u ${env.DOCKER_CREDS_USR} -p ${env.DOCKER_CREDS_PSW}"
        echo "-------------Docker Push-----------"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                
    }
}