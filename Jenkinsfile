pipeline {
    agent { label 'docker' }
    tools {
        maven 'ADOP Maven'
    }
    environment {
        PROJECT_NAME = 'ExampleWorkspace/ExamplePipelineProject'
        WORKSPACE_NAME = 'ExampleWorkspace'
    }
    stages {
        stage('SCM') {
            steps {
                git url: 'ssh://jenkins@gerrit:29418/ExampleWorkspace/ExampleProject/spring-petclinic', credentialsId: 'adop-jenkins-master'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
            }
        }
        stage('Test') {
            steps {
                sh 'mvn clean test'
                junit '**/target/surefire-reports/*.xml'
            }
        }
        stage('Static Code Analysis') {
            steps {
                script {
                    def sonarQubeRunnerHome = tool 'ADOP SonarRunner 2.4'
                    withSonarQubeEnv('ADOP Sonar') {
                        sh "${sonarQubeRunnerHome}/bin/sonar-runner -e -Dsonar.jdbc.url='${SONAR_JDBC_URL}' -Dsonar.host.url='${SONAR_HOST_URL}' -Dsonar.login='${SONAR_LOGIN}' -Dsonar.password='${SONAR_PASSWORD}'"
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            environment {
                ENVIRONMENT_NAME = 'CI'
            }
            steps {
                deleteDir()

                git url: 'ssh://jenkins@gerrit:29418/ExampleWorkspace/ExampleProject/adop-cartridge-java-environment-template', credentialsId: 'adop-jenkins-master'

                script {
                    def SERVICE_NAME="${PROJECT_NAME}".replaceAll('/', '_') + "_${ENVIRONMENT_NAME}"
                    withEnv(["SERVICE_NAME=${SERVICE_NAME}"]) {
                        sh "docker-compose -p ${SERVICE_NAME} up -d"
                        sh "sed -i 's/###TOMCAT_SERVICE_NAME###/${SERVICE_NAME}/' tomcat.conf"
                        sh "docker cp tomcat.conf proxy:/etc/nginx/sites-enabled/${SERVICE_NAME}.conf"

                        step ([$class: 'CopyArtifact', projectName: "${JOB_NAME}", filter: 'target/petclinic.war', selector: [$class: 'SpecificBuildSelector', buildNumber: "${BUILD_NUMBER}"]]);
                        sh "docker cp ${WORKSPACE}/target/petclinic.war ${SERVICE_NAME}:/usr/local/tomcat/webapps/"
                        sh "docker restart ${SERVICE_NAME}"
                    }
                }

                echo "Reload NGINX configuration..."
                sh "docker exec proxy /usr/sbin/nginx -s reload"
            }
        }
        stage('Regression Tests') {
            environment {
                ENVIRONMENT_NAME = 'CI'
            }
            steps {
                parallel(
                    "Cucumber_BDD": {
                        deleteDir()

                        git url: 'ssh://jenkins@gerrit:29418/ExampleWorkspace/ExampleProject/adop-cartridge-java-regression-tests', credentialsId: 'adop-jenkins-master'

                        script {
                            def SERVICE_NAME="${PROJECT_NAME}".replaceAll('/', '_') + "_${ENVIRONMENT_NAME}"
                            withEnv(["SERVICE_NAME=${SERVICE_NAME}"]) {
                                sh "mvn clean -B test -DPETCLINIC_URL=http://${SERVICE_NAME}:8080/petclinic"

                                step([
                                    $class: 'CucumberReportPublisher',
                                    failedFeaturesNumber: 0,
                                    failedScenariosNumber: 0,
                                    failedStepsNumber: 0,
                                    fileExcludePattern: '',
                                    fileIncludePattern: '**/*.json',
                                    jsonReportDirectory: 'target',
                                    parallelTesting: false,
                                    pendingStepsNumber: 0,
                                    skippedStepsNumber: 0,
                                    trendsLimit: 0,
                                    undefinedStepsNumber: 0
                                ])
                            }
                        }
                    },
                    "OWASP_Security": {
                        echo 'testing in owasp'
                    }
                )
            }

        }
        stage('Deployment') {
            steps {
                parallel(
                    "Production_A": {
                      script {
                          def userInput = input(
                              id: 'userInputA',
                              message: 'Are you sure?'
                          )
                      }
                    },
                    "Production_B": {
                        script {
                            def userInput = input(
                                id: 'userInputB',
                                message: 'Are you sure?'
                            )
                        }
                    }
                )
            }
        }
    }
}
