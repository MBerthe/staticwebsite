/* import shared library */

@Library('shared-library') _

pipeline {
  environment {
    IMAGE_NAME="staticwebsite"
    IMAGE_TAG = "latest"
    STAGING = "staticwebsite-mb-staging"
    PRODUCTION="staticwebsite-mb-prod"
    DOCKERHUB_ID = "mnberthe"
    DOCKERHUB_CREDENTIALS_USR = credentials('dockerhub')
    HEROKU_API_KEY  = credentials('heroku_api_key')
    INTERNAL_PORT = "80"
    EXTERNAL_PORT = "80"
    CONTAINER_IMAGE = "${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
  }
  agent any
  
  stages {
    stage('Build image'){
      agent any
      steps {
        script {
          sh 'docker build -t $DOCKERHUB_ID/$IMAGE_NAME:$IMAGE_TAG .'
        }
      }
    }
    stage('Run container based on builded image') {  
      agent any
      steps {
        script {
          sh '''
              echo "Cleaning existing container if exist"
              docker ps -a | grep -i $IMAGE_NAME && docker rm -f $IMAGE_NAME
              docker run -d -p $EXTERNAL_PORT:$INTERNAL_PORT -e PORT=$INTERNAL_PORT --name $IMAGE_NAME  ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
              sleep 5
          '''
         }
      }
    }

    stage('Test image') {
       agent any
       steps {
          script {
            sh '''
               curl -v 172.17.0.1:$EXTERNAL_PORT | grep -i "Dimension"
            '''
          }
       }
    }

    stage('Clean container') {
      agent any
      steps {
         script {
           sh '''
               docker stop $IMAGE_NAME
               docker rm $IMAGE_NAME
           '''
         }
      }
    }
  
    stage ('Login and Push Image on docker hub') {
      agent any
      steps {
         script {
           sh '''
               echo $DOCKERHUB_ID | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
               docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
           '''
         }
      }
    }
    
    stage ('STAGING - Deploy') {
      agent any
      environment{
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
         script {
           sh '''
               
                heroku container:login
                heroku create $STAGING || echo "project already exist"
                heroku container:push -a $STAGING web
                heroku container:release -a $STAGING web
           '''
         }
      }
    }
    stage ('PROD - Deploy') {
      agent any
      when {
        expression { GIT_BRANCH == 'origin/main' }
      }
      environment{
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
         script {
           sh '''
               
                heroku container:login
                heroku create $PRODUCTION || echo "project already exist"
                heroku container:push -a $PRODUCTION web
                heroku container:release -a $PRODUCTION web
           '''
         }
      }
    }

  }
  
  post {
    always {
      script {
        slackNotifier currentBuild.result
      }
    }
  }
} 
