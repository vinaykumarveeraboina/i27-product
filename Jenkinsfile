// Call the Shared Lib

// This Jenkinsfile is for user deployment

pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'buildOnly', choices: 'NO\nYES', description: "This will only build the application")
        choice(name: 'scanOnly', choices: 'NO\nYES', description: "This will only SCAN the application")
        choice(name: 'dockerpush', choices: 'NO\nYES', description: "This will build the application, Docker Build, and Docker push")
        choice(name: 'deployToDev', choices: 'NO\nYES', description: "This will deploy the app to Dev")
        choice(name: 'deployToTest', choices: 'NO\nYES', description: "This will deploy the app to Test")
        choice(name: 'deployToStage', choices: 'NO\nYES', description: "This will deploy the app to Stage")
    }
    options {
        buildDiscarder(logRotator(daysToKeepStr: '7', numToKeepStr: '5'))
    }
    environment {
        DOCKERHUB = "docker.io/vinaykumarveeraboina"
        APPLICATION_NAME = 'product'
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_CREDS = credentials('DockerHub')
        SONAR_URL = 'http://35.224.218.172:9000'
        SONAR_TOKEN = credentials('sonar-product')
    }
    tools {
        maven 'maven-3.8.8'
        jdk 'Jdk17'
    }
    stages {
        stage('Build') {
            when {
                anyOf {
                    expression { params.buildOnly == 'YES' }
                    expression { params.dockerpush == 'YES' }
                }
            }
            steps {
                script {
                    applicationBuild()
                }
            }
        }
        stage('Unit-Test') {
            when {
                anyOf {
                    expression { params.buildOnly == 'YES' }
                    expression { params.dockerpush == 'YES' }
                }
            }
            steps {
                echo "Testing the ${env.APPLICATION_NAME} application"
                sh "mvn test"
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Sonar_Test') {
            when {
                anyOf {
                    expression { params.scanOnly == 'YES' }
                }
            }
            steps {
                echo " ************************* STARTING SONAR ANALYSIS with Quality gate ************************"
                withSonarQubeEnv('SonarQube') {
                    sh """
                    mvn clean verify sonar:sonar \
                     -Dsonar.projectKey=i27-product \
                     -Dsonar.host.url=${env.SONAR_URL} \
                     -Dsonar.login=${SONAR_TOKEN}
                    """
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        /*stage('Docker-Format') {
            steps {
                echo "ACTUAL_FORMAT: ${APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING}"
                echo "CUSTOM_FORMAT: ${APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${POM_PACKAGING}"
            }
        }*/
        stage('Docker Build and Push') {
            when {
                anyOf {
                    expression { params.dockerpush == 'YES' }
                }
            }
            steps {
                script {
                    dockerBuildPush()
                }
            }
        }
        stage('Docker deploy to DEV') {
            when {
                anyOf {
                    expression { params.deployToDev == 'YES' }
                }
            }
            steps {
                script {
                    imagevalidation()
                    DockerDeploy('dev', '5132', '8132')
                }
            }
        }
        stage('Docker deploy to TEST env') {
            when {
                anyOf {
                    expression { params.deployToTest == 'YES' }
                }
            }
            steps {
                script {
                    imagevalidation()
                    DockerDeploy('test', '6132', '8132')
                }
            }
        }
        stage('Docker deploy to STAGE env') {
            when {
              allOf{
                  //this will execute when the branch is release and deployToStage==yes 

                anyOf {
                    expression { params.deployToStage == 'YES' }
                }
                anyOf {
                     branch 'release/*' 
                }
            }
            }
            steps {
               timeout( time :300, unit :'SECONDS'){
               input message : " Deploying  ${env.APPLICATION_NAME} to stage ??? " , ok :'YES', submitter : 'owner'
               }
                script {
                    imagevalidation()
                    DockerDeploy('stage', '7132', '8132')
                }
            }
        }

        stage (' Clean Workspace ') {

          steps{
            cleanWs()
          }
        }
    }
}

// This method is developed for deploying our app in different environments
def DockerDeploy(envdeploy, hostport, contport) {
    echo "************************ Deploying to Docker $envdeploy ********************************"

    withCredentials([usernamePassword(credentialsId: 'dockerdev', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
        echo "****************** PULLING the container from docker hub ********************"
        sh """
        sshpass -p ${env.PASSWORD} ssh -o StrictHostKeyChecking=no ${env.USERNAME}@${env.docker_dev_server} docker pull ${env.DOCKERHUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
        """
        script {
            try {
                echo "****************** Stopping the container ********************"
                sh """
                sshpass -p ${env.PASSWORD} ssh -o StrictHostKeyChecking=no ${env.USERNAME}@${env.docker_dev_server} docker stop ${env.APPLICATION_NAME}-$envdeploy
                """

                echo "****************** Removing the container ********************"
                sh """
                sshpass -p ${env.PASSWORD} ssh -o StrictHostKeyChecking=no ${env.USERNAME}@${env.docker_dev_server} docker rm ${env.APPLICATION_NAME}-$envdeploy
                """
            } catch (err) {
                echo "Caught the Error: ${err}"
            }
        }

        echo "**************** Running the container *****************"
        sh """
        sshpass -p ${env.PASSWORD} ssh -o StrictHostKeyChecking=no ${env.USERNAME}@${env.docker_dev_server} docker run -d -p $hostport:$contport --name ${env.APPLICATION_NAME}-$envdeploy ${env.DOCKERHUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
        """
    }
}

def imagevalidation() {
    println(" **********************  pulling the docker image *******************************")
    try {
        sh """
        docker pull ${env.DOCKERHUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
        """
    } catch (Exception e) {
        println( " *******************   OOPS! Image with this tag is not available  ************************* ")
        println("*********************** Building Application  *****************************************")
        applicationBuild()
        println("*********************** Image build and push to Hub  *****************************************")
        dockerBuildPush()
    }
}

// This method will build the application
def applicationBuild() {
    echo "Building the ${env.APPLICATION_NAME} application"
    sh "mvn clean package -DskipTests=true"
}

// This function will build the image and push to docker hub
def dockerBuildPush() {
    sh """
    ls -la
    cp ${workspace}/target/i27-${APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING} ./.cicd
    ls -la ./.cicd
    echo "*********************** Building Docker Image *********************************"
    docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=i27-${APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING} -t ${env.DOCKERHUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd
    docker images
    echo "***************** Docker login ************************"
    docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
    echo "********************* Docker push *************************************"
    docker push ${env.DOCKERHUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
    """
}
