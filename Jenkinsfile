
pipeline {
  environment {
    registry = "rohammosalli/app1"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') { //this part wll pull our source code 
      steps {
        git 'https://github.com/rohammosalli/python-app.git'
      }
    }
    stage('Building image') { // this part will build our Docker Image
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') { // after build image will push it to Docker registry , don't forget setup credential in Jenkins

      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') { // this will remove Unused Image in Jenkins Host
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }


    stage('helm deploy') { // this part will be deploy our helm chart
      steps{ 
        sh "helm list"
        sh "helm upgrade --namespace core --install python-app1 ./deploy-app1 --wait --force --set image=rohammosalli/app1:${BUILD_NUMBER}"
        sh "htpasswd -b -c password username password" 
        sh "kubectl create secret generic basic-auth --from-file=password"
        sh "kubectl -n b2c create secret generic basic-auth  --from-file=password --dry-run=true -o yaml | kubectl apply -f -"
      
      }
    }
  }
} 