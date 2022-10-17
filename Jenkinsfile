node('docker-slave') {
 stage('Poll') {
   checkout scm
 }
 stage('Build & Unit test'){
    sh 'mvn clean verify -DskipITs=true';
    step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml', healthScaleFactor: 1.0])
    junit '**/target/surefire-reports/TEST-*.xml'
 }
  stage('Static Code Analysis'){
   withSonarQubeEnv(installationName: 'Sonarqube'){
      sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project -Dsonar.projectKey=example-project -Dsonar.projectVersion=$BUILD_NUMBER';
   }
  }
  stage ('Integration Test'){
    sh 'mvn clean verify -Dsurefire.skip=true';
    junit '**/target/failsafe-reports/TEST-*.xml'
    archive 'target/*.jar'
 }
 stage ('Publish'){
   def server = Artifactory.server 'Default Artifactory Server'
   def uploadSpec = """{
      "files": [
       {
          "pattern": "target/hello-0.0.1.war",
          "target": "example-project/${BUILD_NUMBER}/ ",
          "props": "Integration-Tested=Yes;Performance-Tested=No"
       }
     ]
   }"""
   server.upload(uploadSpec)
 }
 stash includes: 'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name:'binary'
}

node('docker_pt') {
 stage ('Start Tomcat'){
   sh '''cd /home/jenkins/tomcat/bin /startup.sh''';
 }
 stage ('Deploy '){
   unstash 'binary'
    sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/';
 }
 stage ('Performance Testing'){
   sh '''cd /opt/jmeter/bin/ ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl''';
    archiveArtifacts '**/*.jtl'
  }
 stage ('Promote build in Artifactory'){
   withCredentials([usernameColonPassword(credentialsId:
      'artifactory-account', variable: 'credentials')]) {
        sh 'curl -u${credentials} -X PUT "http://172.17.0.1:8082/artifactory/api/storage/example-project/ ${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes"';
     }
 }
}
