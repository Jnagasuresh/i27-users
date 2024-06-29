// this jenkinsfile is for Eureka microservice

pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice (name: 'buildonly',
                choices: 'no\nyes',
                description: "build the applicaiton only"
        )
         choice (name: 'scanonly',
                choices: 'no\nyes',
                description: "scan the applicaiton only"
        )
         choice (name: 'dockerPush',
                choices: 'no\nyes',
                description: "This will build the app, push to registry"
        )
        choice (name: 'deployToDev',
                choices: 'no\nyes',
                description: "This will deploy the app to Dev env"
        )
        choice (name: 'deployToTest',
                choices: 'no\nyes',
                description: "This will deploy the app to Test env"
        )
        choice (name: 'deployToStage',
                choices: 'no\nyes',
                description: "This will deploy the app to Stage env"
        )
        choice (name: 'deployToProd',
                choices: 'no\nyes',
                description: "This will deploy the app to Prod env"
        )
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    environment{
        APPLICATION_NAME = "user"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        SONAR_URL = "http://34.73.212.154:9000"
        SONAR_TOKEN = credentials('sonar_creds')
        DOCKER_HUB ="docker.io/jnagasuresh"
        DOCKER_CREDS = credentials('docker_creds')
    }
    stages {
        stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.buildonly == 'yes'
                    }
                }
            }
            // This step will take care of building the application
            steps {
                script {
                    buildApp().call()
                }
            }
        }
      
        stage ('sonar') {
            // sqa_3aa5ca02bb917b5a93a398c021068ba311697d5c
            when {
                    anyOf {
                        expression {
                            params.scanonly == 'yes'
                        }
                    }
                }
            steps {
                echo "StartingSonar Scans with QualityGates"
                // Before we go to tnext step, install sonarqube plugin
                // next goto manage jenkins > configure > sonarquebe > give url and token for sonar
                withSonarQubeEnv('SonarQube'){ // SonarQube is the name that we configured in manage jenkins > Sonarqube
                    sh """
                    echo "Starting Sonar scan"
                    mvn sonar:sonar \
                        -Dsonar.projectKey=i27-user \
                        -Dsonar.host.url=${env.SONAR_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
                timeout (time:2, unit:'MINUTES'){
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        
        }
        stage ("Docker Build and Push") {
            when {
                    anyOf {
                        expression {
                           // params.buildonly == 'yes'
                            params.dockerPush == 'yes'
                        }
                    }
                }
             steps {
                script {
                   dockerBuildandPush().call()
                }
            }
       
        }
        stage ('Deploy To Dev')
        {
             when {
                    anyOf {
                        expression {
                            params.deployToDev == 'yes'
                        }
                    }
                }
             steps {
                script {
                    imageValidation().call()
                    dockerDeploy('dev','5232', '8232').call()
                    echo "deployed to dev environment!!"
                }
             }
          
        }
         stage ('Deploy To Test')
        {
             when {
                    anyOf {
                        expression {
                            params.deployToTest == 'yes'
                        }
                    }
                }
             steps {
                script {
                    imageValidation().call()
                    dockerDeploy('test','6232', '8232').call()
                    echo "deployed to test environment successfull!!"
                }
             }
            
        }
         stage ('Deploy To stage ')
        {
             when {
                    anyOf {
                        expression {
                            params.deployToStage == 'yes'
                        }
                    }
                }
             steps {
                script {
                    imageValidation().call()
                    dockerDeploy('stage','7232', '8232').call()
                    echo "deployed to stage environment successfull!!"
                }
             }
        }
         stage ('Deploy To Prod ')
        {
             when {
                 allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                            // other condition as well
                        }
                    }
                    anyOf {
                        expression {
                            branch 'release/*'
                        }
                    }
                 }
                }
             steps {
                timeout(time: 300, unit: 'SECONDS') {
                    input message: "Deploying to ${env.APPLICATION_NAME} TO production ???", ok: 'yes', submitter: 'mat'
                }
                script {
                    imageValidation().call()
                    dockerDeploy('stage','8232', '8232').call()
                    echo "deployed to Prod environment successfull!!"
                }
             }
        }
    }
}

def imageValidation() {
    return {
        println ("Pulling the Docker image")
        try {
          sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
        catch (Exception e) {
            println("OOPS!!!!!, docker image with this tag doesnot exists, So creating the image")
            buildApp().call()
            dockerBuildandPush().call()
        }
    }
}

def buildApp() {
    return {
         echo "Building the ${env.APPLICATION_NAME} Application"
        //mvn command 
        sh 'mvn clean package -DskipTests=true'
        archiveArtifacts artifacts: 'target/*.jar'
    }
}

def dockerBuildandPush() {
    return {
         echo "Starting Docker build stage"
        sh "cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd/"
        echo "**************************** Building Docker Image ****************************"
        sh "docker build --force-rm --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd"        
        echo "**************************** Login to Docke Repo ****************************"
        sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        echo "**************************** Docker Push ****************************"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"

    }
}

def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
        // for every env what will change ?????
        // application name, hostport, container port, container name, environment
        echo "**************************** Deploying to $envDeploy Environment ****************************"
        withCredentials([usernamePassword(credentialsId: 'maha_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
            script {
                // Pull the image on the docker server
                sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                try {
                    // Stop the container
                    sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"
                    // Remove the Container
                    sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
                } catch(err) {
                    echo "Error Caught: $err"
                }
                // Create the container
                echo "Creating the Container"
                sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            }

        }
    }
}

/*
 there are 3 ways sonar implementation can be done
 1. pom.xml ---> Sonar properties
 2. sonar.properties ---> with sonar properties
 3. own method ----> with sonar properties

*/

//Sonar scan command
// mvn sonar:sonar user/password or token and where my sonar (host url) is and project key