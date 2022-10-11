# Java-spring-tomcat-application CI-CD

In this project we are going a CI-CD pipeline on ```Jenkins``` to deploy ```Java Spring``` Application on a ```Tomcat``` container using ```Kubernetes```.

## Tools used to build pipeline

- [Jenkins](https://www.jenkins.io/) - To build CI-CD pipeline
- [Gradle](https://gradle.org/) - To build the Java package 
- [SonarQube](https://www.sonarqube.org/) - To do static Code Analysis
- [Nexus](https://www.sonatype.com/) - To store the Artifacts
- [Docker](https://www.docker.com/) - To build and push docker image 
- [Datree](https://www.datree.io/) - To validate the helm charts
- [Kubernetes](https://kubernetes.io/) - To deploy the Java application on Tomcat container

## Pipeline stages

- Sonar Quality Check
```json
stage("Sonar Quality Check"){
    steps{
        script{
            withSonarQubeEnv('sonar-server') {
                sh '''
                chmod +x gradlew
                ./gradlew sonarqube
                '''
            }
            timeout(30) {
                def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
            }
        }  
    }
}
```


