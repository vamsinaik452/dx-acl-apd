pipeline {

  environment {
    devRegistry = 'ghcr.io/datakaveri/acl-apd-dev'
    deplRegistry = 'ghcr.io/datakaveri/acl-apd-depl'
    testRegistry = 'ghcr.io/datakaveri/acl-apd-test:latest'
    registryUri = 'https://ghcr.io'
    registryCredential = 'datakaveri-ghcr'
    GIT_HASH = GIT_COMMIT.take(7)
  }

  agent { 
    node {
      label 'slave1' 
    }
  }

  stages {
    stage('Conditional Execution') {
      when {
        anyOf {
          triggeredBy 'UserIdCause'
          expression {
            def comment = env.ghprbCommentBody
            return comment && comment != "null" && !comment.trim().isEmpty()
          }
          changeset "docker/**"
          changeset "docs/**"
          changeset "pom.xml"
          changeset "src/main/**"
        }
      }
      stages {
        stage('Trivy Code Scan (Dependencies)') {
          steps {
            script {
              sh '''
                trivy fs --scanners vuln,secret,misconfig --output trivy-fs-report.txt .
              '''
            }
          }
        }

        stage('Building images') {
          steps{
            script {
              echo 'Pulled - ' + env.GIT_BRANCH
              devImage = docker.build( devRegistry, "-f ./docker/dev.dockerfile .")
              deplImage = docker.build( deplRegistry, "-f ./docker/depl.dockerfile .")
            }
          }
        }

        stage('Trivy Docker Image Scan and Report') {
          steps {
            script {
              sh "trivy image --output trivy-dev-image-report.txt ${devImage.imageName()}"
              sh "trivy image --output trivy-depl-image-report.txt ${deplImage.imageName()}"
            }
          }
          post {
            always {
              archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true
              publishHTML(target: [
                allowMissing: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'trivy-fs-report.txt, trivy-dev-image-report.txt, trivy-depl-image-report.txt',
                reportName: 'Trivy Reports'
              ])
            }
          }
        }

        stage('Unit Tests and CodeCoverage Test'){
          steps{
            script{
              sh 'sudo update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java'
              sh 'mkdir -p configs'
              sh 'cp /home/ubuntu/configs/apd-config-test.json ./configs/config-dev.json'
              sh 'mvn clean test -Dmaven.test.failure.ignore=true checkstyle:checkstyle pmd:pmd'
            }
            xunit (
              thresholds: [ skipped(failureThreshold: '25'), failed(failureThreshold: '5') ],
              tools: [ JUnit(pattern: 'target/surefire-reports/*.xml') ]
            )
            jacoco classPattern: 'target/classes', execPattern: 'target/jacoco.exec', sourcePattern: 'src/main/java', exclusionPattern: '**/*VertxEBProxy.class, **/*VertxProxyHandler.class, **/*Verticle.class, **/*Service.class, iudx/apd/acl/server/deploy/*, **/*Constants.class'
          }
          post{
            always {
              recordIssues(
                enabledForFailure: true,
                skipBlames: true,
                qualityGates: [[threshold:0, type: 'TOTAL', unstable: false]],
                tool: checkStyle(pattern: 'target/checkstyle-result.xml')
              )
              recordIssues(
                enabledForFailure: true,
                skipBlames: true,
                qualityGates: [[threshold:0, type: 'TOTAL', unstable: false]],
                tool: pmdParser(pattern: 'target/pmd.xml')
              )
            }
            failure{
              xunit (
                thresholds: [ skipped(failureThreshold: '25'), failed(failureThreshold: '5') ],
                tools: [ JUnit(pattern: 'target/surefire-reports/*.xml') ]
              )
              error "Test failure. Stopping pipeline execution!"
            }
            cleanup{
              script{
                sh 'sudo update-alternatives --set java /usr/lib/jvm/java-11-openjdk-amd64/bin/java'
                sh 'sudo rm -rf target/'
              }
            }
          }
        }

        stage('Start acl-apd-Server for Integration Testing'){
          steps{
            script{
              sh 'sudo update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java'
              sh 'scp src/test/resources/DX-ACL-APD-APIs.postman_collection.json jenkins@jenkins-master:/var/lib/jenkins/iudx/acl-apd/Newman/'
              sh 'mvn flyway:migrate -Dflyway.configFiles=/home/ubuntu/configs/acl-apd-flyway.conf'
              sh 'docker compose -f docker-compose.test.yml up -d integTest'
              sh 'sleep 45'
            }
          }
          post{
            failure{
              script{
                sh 'mvn flyway:clean -Dflyway.configFiles=/home/ubuntu/configs/acl-apd-flyway.conf'
                sh 'sudo update-alternatives --set java /usr/lib/jvm/java-11-openjdk-amd64/bin/java'
                sh 'docker compose -f docker-compose.test.yml down --remove-orphans'
              }
              cleanWs deleteDirs: true, disableDeferredWipeout: true
            }
          }
        }

        stage('Integration Tests and OWASP ZAP pen test'){
          steps{
            node('built-in') {
              script{
                startZap ([host: 'localhost', port: 8090, zapHome: '/var/lib/jenkins/tools/com.cloudbees.jenkins.plugins.customtools.CustomTool/OWASP_ZAP/ZAP_2.11.0'])
                sh 'curl http://127.0.0.1:8090/JSON/pscan/action/disableScanners/?ids=10096'
                sh 'HTTP_PROXY=\'127.0.0.1:8090\' newman run /var/lib/jenkins/iudx/acl-apd/Newman/DX-ACL-APD-APIs.postman_collection.json -e /home/ubuntu/configs/acl-apd-postman-env.json --insecure -r htmlextra --reporter-htmlextra-export /var/lib/jenkins/iudx/acl-apd/Newman/report/report.html --reporter-htmlextra-skipSensitiveData'
                runZapAttack()
              }
            }
          }
          post{
            always{
              node('built-in') {
                script{
                  publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '/var/lib/jenkins/iudx/acl-apd/Newman/report/', reportFiles: 'report.html', reportTitles: '', reportName: 'Integration Test Report'])
                  archiveZap failHighAlerts: 1, failMediumAlerts: 1, failLowAlerts: 1
                }
              }
            }
            failure{
              error "Test failure. Stopping pipeline execution!"
            }
            cleanup{
              script{
                sh 'mvn flyway:clean -Dflyway.configFiles=/home/ubuntu/configs/acl-apd-flyway.conf'
                sh 'sudo update-alternatives --set java /usr/lib/jvm/java-11-openjdk-amd64/bin/java'
                sh 'docker compose -f docker-compose.test.yml down --remove-orphans'
              }
            }
          }
        }
        
        stage('Continuous Deployment') {
          when {
              expression {
                return env.GIT_BRANCH == 'origin/main';
              }
            }
          stages {
            stage('Push Images') {
              steps {
                script {
                  docker.withRegistry( registryUri, registryCredential ) {
                    devImage.push("1.2.0-alpha-${env.GIT_HASH}")
                    deplImage.push("1.2.0-alpha-${env.GIT_HASH}")
                  }
                }
              }
            }
            stage('Docker Swarm deployment') {
              steps {
                script {
                  sh "ssh azureuser@docker-swarm 'docker service update acl-apd_acl-apd --image ghcr.io/datakaveri/acl-apd-depl:1.2.0-alpha-${env.GIT_HASH}'"
                  sh 'sleep 40'
                  sh '''#!/bin/bash
                    response_code=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 --retry 5 --retry-connrefused -XGET https://acl-apd.iudx.io/apis)
                    echo $response_code
                    if [[ "$response_code" -ne "200" ]]
                    then
                      echo "Health check failed"
                      exit 1
                    else
                      echo "Health check complete; Server is up."
                      exit 0
                    fi
                  '''
                }
              }
              post{
                failure{
                  error "Failed to deploy image in Docker Swarm"
                }
              }
            }
          }
        }

      }
    }
  }
      post{
        failure{
          script{
            if (env.GIT_BRANCH == 'origin/main')
            emailext recipientProviders: [buildUser(), developers()], to: '$AAA_RECIPIENTS, $DEFAULT_RECIPIENTS', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', body: '''$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
    Check console output at $BUILD_URL to view the results.'''
          }
        }
      }
    }