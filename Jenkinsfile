node ('Slave'){
  String jdktool = tool name: "jdk8", type: 'hudson.model.JDK'
  def mvnHome = tool name: 'maven'
  List javaEnv = [
    "PATH+MVN=${jdktool}/bin:${mvnHome}/bin",
    "M2_HOME=${mvnHome}",
    "JAVA_HOME=${jdktool}"
  ]
  withEnv(javaEnv) {
    stage('Clear & Checkout') {
      cleanWs()
      git url: 'https://github.com/vdt-mik/DevOps028-Demo3'
    }
    stage('Test & Build') {
      try {
      sh 'mvn clean test'
      sh 'mvn clean package'
      } catch (e) {
          currentBuild.result = 'FAILURE'
        }    
    }
    stage ('Post') {
      if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
        archiveArtifacts artifacts: 'target/*.jar', onlyIfSuccessful: true  
        sh 'tar -czf liquibase.tar.gz liquibase'
        sh 'cp target/Samsara-*.jar .'
        sh 'aws s3 cp target/Samsara-*.jar s3://mik-bucket/'
        sh 'aws s3 cp liquibase.tar.gz s3://mik-bucket/'
        sh 'aws s3 cp aws/user-data.sh s3://mik-bucket/'
      }
    }
  }           
  stage('Deploy RDS') {
    sh 'chmod +x aws/rds.sh && ./aws/rds.sh'
  }
  stage('Build docker image') {
    DB_HOST = sh(
      script: "aws ssm get-parameters --names DB_HOST --with-decryption --output text | awk '{print \$4}'",
      returnStdout: true
      ).trim()
    DB_PORT = sh(
      script: "aws ssm get-parameters --names DB_PORT --with-decryption --output text | awk '{print \$4}'",
      returnStdout: true
      ).trim()
    DB_NAME = sh(
      script: "aws ssm get-parameters --names DB_NAME --with-decryption --output text | awk '{print \$4}'",
      returnStdout: true
      ).trim()
    DB_USER = sh(
      script: "aws ssm get-parameters --names DB_USER --with-decryption --output text | awk '{print \$4}'",
      returnStdout: true
      ).trim()
    DB_PASS = sh(
      script: "aws ssm get-parameters --names DB_PASS --with-decryption --output text | awk '{print \$4}'",
      returnStdout: true
      ).trim()
    ART_NAME = sh(
      script: "ls ${WORKSPACE}/target | grep jar | grep -v original",
      returnStdout: true
      ).trim()
    def samsaraImage = docker.build("303036157700.dkr.ecr.eu-central-1.amazonaws.com/samsara:samsara-${env.BUILD_ID}","--build-arg DB_HOST=${DB_HOST} --build-arg DB_PORT=${DB_PORT} --build-arg DB_NAME=${DB_NAME}, " +
                                    "--build-arg DB_USER=${DB_USER} --build-arg DB_PASS=${DB_PASS} --build-arg ART_NAME=${ART_NAME} .")
  }

  stage('build & push docker image') {
    docker.withRegistry('https://303036157700.dkr.ecr.eu-central-1.amazonaws.com', 'ecr:eu-central-1:ceb0ba5d-18be-4d4c-8090-1120568d9a14') {
      docker.image("303036157700.dkr.ecr.eu-central-1.amazonaws.com/samsara:samsara-${env.BUILD_ID}").push()
    }
  }




//    docker.withRegistry() {
//    sh 'pwd_22=`aws ecr get-login --no-include-email --region eu-central-1 | awk \'{print \$6}\'` && docker login -u AWS -p "${pwd_22}" https://303036157700.dkr.ecr.eu-central-1.amazonaws.com/samsara'
//    def samsaraPush = docker.image("samsara-${env.BUILD_ID}").push()
//    docker.withRegistry('https://303036157700.dkr.ecr.eu-central-1.amazonaws.com', '') {
//samsara:${env.BUILD_ID}
//        def samsaraImage = docker.build("samsara:${env.BUILD_ID}")

        /* Push the container to the custom Registry */
//        samsaraImage.push()
//    }
//  }
  stage('Deploy ASG') {
    sh 'chmod +x aws/asg.sh && ./aws/asg.sh'
  }  
  stage('Check APP') {
    timeout(time: 1, unit: 'MINUTES') {
      waitUntil {
        try {
          APP_URI = sh(
          script: "aws ssm get-parameters --names APP_URL --with-decryption --output text | awk '{print \$4}'",
          returnStdout: true
          ).trim()
          def response = httpRequest "http://$APP_URI/login" 
          println("Status: "+response.status) 
          println("Content: "+response.content)
          return true
        } catch (Exception e) {
            return false
          }
      }
    }     
  } 
}