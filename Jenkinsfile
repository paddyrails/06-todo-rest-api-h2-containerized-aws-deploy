pipeline {
    agent any
    options {
    	withAWS(profile:'default')
    }
    environment {
        PATH="/usr/local/bin:$PATH"
        DOCKER_HUB_CREDENTIALS= credentials('DOCKER_HUB_CREDENTIALS')     
    }
    parameters {
      string(defaultValue: "1.0.0.$BUILD_NUMBER", description: "Release Version", name: "releaseVersion")
      string(defaultValue: "todo-rest-api-h2", description: "Release ArtifactId", name: "releaseArtifactId")
      string(defaultValue: "Todo-rest-api-docker-$BUILD_NUMBER", description: "New Version to be deployed", name: "versionLabel")
      string(defaultValue: "todo-rest-api", description: "Application Name", name: "applicationName")
      string(defaultValue: "todo-rest-api-template-$BUILD_NUMBER", description: "Temlate Name", name: "templateName")
      string(defaultValue: "Todo-rest-api-docker-dev", description: "Env Name", name: "environmentName")
      string(defaultValue: "e-ugrdmgk2nm", description: "Current Env id", name: "environmentId")
      string(defaultValue: "Todo-rest-api-docker-$BUILD_NUMBER-blue", description: "New Blue Env Name", name: "blueEnvironmentName")

    }
    stages {
        stage('Build') {
            steps {
                echo "BUILD"
                sh "mvn clean package -Drelease.artifactId=${params.releaseArtifactId} -Drelease.version=${params.releaseVersion}"                
            }
        }
        stage('Test') {
            steps {
                echo "Test"
            }            
        }
        stage('Login') {
          steps {
            sh "docker login -u $DOCKER_HUB_CREDENTIALS_USR -p $DOCKER_HUB_CREDENTIALS_PSW"
          }
        }
        stage('Push Docker image and Upload artifact to s3') {         
            steps {
                sh "docker push paddypillai/${params.releaseArtifactId}:${params.releaseVersion}"
                script {
                  jsonfile = readJSON file: 'Dockerrun.aws.json', returnPojo: true
                  jsonfile['Image']['Name'] = "paddypillai/${params.releaseArtifactId}:${params.releaseVersion}"
                  writeJSON file: 'Dockerrun.aws.json', json: jsonfile
                }
                // upload updated Dockerrun.aws.json
                s3Upload(file:"Dockerrun.aws.json", 
                bucket:"popsy-bucket", 
                path:"Todo-rest-api-docker/$BUILD_NUMBER/Dockerrun.aws.json")
            }            
        }
        stage('Deliver') {
            steps {
                sh 'eb --version'
                ebCreateApplicationVersion(
                    applicationName: "${params.applicationName}",
                    versionLabel: "${params.versionLabel}",
                    s3Bucket: "popsy-bucket",
                    s3Key: "Todo-rest-api-docker/$BUILD_NUMBER/Dockerrun.aws.json",
                    description: "New version"
                )
                // Create configuration template based on existing environment
                ebCreateConfigurationTemplate(
                    applicationName: "${params.applicationName}",
                    templateName: "${params.templateName}",
                    environmentId:  "${params.environmentId}",,
                    description: "Configuration template for todo-rest-api-docker"
                )
                // Create environment from existing configuration template - New version
                ebCreateEnvironment(
                   applicationName: "${params.applicationName}",
                    environmentName: "${params.blueEnvironmentName}",
                    templateName: "${params.templateName}",
                    versionLabel: "${params.versionLabel}",
                    description: "Blue environment"
                )
                // Wait for environment health to be green for at least 1 minute
                ebWaitOnEnvironmentHealth(
                    applicationName: "${params.applicationName}",
                    environmentName: "${params.blueEnvironmentName}",
                )
                // Swap CNAMEs using the environment names
                ebSwapEnvironmentCNAMEs(
                    sourceEnvironmentName: "${params.blueEnvironmentName}",
                    destinationEnvironmentName: "${params.environmentName}",
                )
            }
        }
        stage('Complete') {
            steps {
                echo 'Job Complete!'
            }
        }
    }
    post {
      always {
        sh 'docker logout'
      }
    }
}
