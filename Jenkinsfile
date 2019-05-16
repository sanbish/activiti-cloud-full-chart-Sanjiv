pipeline {
    options {
      disableConcurrentBuilds()
    }  
    agent {
      label "jenkins-maven-java11"
    }
    environment {
      ORG               = 'activiti'
      APP_NAME          = 'activiti-cloud-full-example'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      GITHUB_CHARTS_REPO    = "https://github.com/Activiti/activiti-cloud-helm-charts.git"
      GITHUB_HELM_REPO_URL = "https://activiti.github.io/activiti-cloud-helm-charts/"
      HELM_RELEASE_NAME = "example-$BRANCH_NAME-$BUILD_NUMBER".toLowerCase()

      PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
      PREVIEW_NAMESPACE = "example-$BRANCH_NAME-$BUILD_NUMBER".toLowerCase()
      GLOBAL_GATEWAY_DOMAIN="35.228.195.195.nip.io"
      REALM = "activiti"
      GATEWAY_HOST = "activiti-cloud-gateway.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"
      SSO_HOST = "activiti-cloud-gateway.$PREVIEW_NAMESPACE.$GLOBAL_GATEWAY_DOMAIN"

    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        steps {
          container('maven') {
          sh 'make validate'
           dir ("./charts/$APP_NAME") {
             // sh 'make build'
              sh 'make install'
            }

            dir("./activiti-cloud-acceptance-scenarios") {
              git 'https://github.com/Activiti/activiti-cloud-acceptance-scenarios.git'
              sh 'sleep 30'
              sh "mvn clean install -DskipTests && mvn -pl 'runtime-acceptance-tests' clean verify"
            }
          }
        }
      }
      stage('Prepare') {
        when {
          branch 'master'
        }
        steps {
          container('maven') {
          // ensure we're not on a detached head
           sh "git checkout master"
           sh "git config --global credential.helper store"
           sh "jx step git credentials"
           sh "echo \$(jx-release-version) > VERSION"
           dir ("./charts/$APP_NAME") {
            sh 'make build'
           }
          }
        }
      }
      stage('Tests') {
        when {
          branch 'master'
        }
        parallel{
            stage ('mvn clean install') {
              steps {
                container('maven') {
                  dir("./activiti-cloud-acceptance-scenarios") {
                  git 'https://github.com/Activiti/activiti-cloud-acceptance-scenarios.git'
                  sh 'mvn clean install -DskipTests'
                 }                
                }
              }
            }
            stage ('rbtest') {
              steps {
                container('maven') {
                   //RUNTIME bundle tests
                  dir ("./charts/$APP_NAME") {
                   // sh 'make build'
                   sh 'make install'
                   }
      //run RB and modeling tests
                   dir("./activiti-cloud-acceptance-scenarios") {
                     // git 'https://github.com/Activiti/activiti-cloud-acceptance-scenarios.git'
                     sh  "mvn -pl 'runtime-acceptance-tests,modeling-acceptance-tests' verify"
                   }    
      //end run RB and modeling tests
                }
                // post{
                //   dir ("./charts/$APP_NAME") {
                //     sh 'make delete'
                //   }
                // }
             }
           }
            stage('run_security')  {
                environment {
                  GATEWAY_HOST = "activiti-cloud-gateway.$PREVIEW_NAMESPACE-security.$GLOBAL_GATEWAY_DOMAIN"
                  SSO_HOST = "activiti-cloud-gateway.$PREVIEW_NAMESPACE-security.$GLOBAL_GATEWAY_DOMAIN" 
                } 
                steps { 
                  container('maven') {
            //SECURITY tests
                  dir ("./charts/$APP_NAME") {
                    sh 'make install-security'
                  }
         //run SECURITY tests
                  dir("./activiti-cloud-acceptance-scenarios") {
                    // git 'https://github.com/Activiti/activiti-cloud-acceptance-scenarios.git'
                    sh "mvn -pl 'security-policies-acceptance-tests' verify"
                   }
                } 
                // post {
                //     always {
                //       dir ("./charts/$APP_NAME") {
                //        sh 'make delete-security'
                //       }
                //     }
                // }
              }
            }
          } 
        }
           
        stage('tag and github') {
             when {
               branch 'master'
             }
             steps {
              container('maven') {   
               dir ("./charts/$APP_NAME") {
                 retry(5) {
                  sh 'make tag'
                }
                 sh 'make release'
                 retry(5) {
                 sh 'make github'
               }
             }
           }
        }
      }
        
       stage('Promote to Environments') {
         when {
            branch 'master'
         }
         steps {
           container('maven') {
             dir ("./charts/$APP_NAME") {
               sh 'jx step changelog --version v\$(cat ../../VERSION)'
              retry(5) {  
                sh 'jx promote -b --all-auto --helm-repo-url=$GITHUB_HELM_REPO_URL --timeout 1h --version \$(cat ../../VERSION) --no-wait'
               }         
             }
           }
         }
       } 
   }
   post {
        always {
          container('maven') {
           dir("./charts/$APP_NAME") {
              sh "make delete"
              sh 'make delete-security'
           }
            sh "kubectl delete namespace $PREVIEW_NAMESPACE" 
            sh "kubectl delete namespace $PREVIEW_NAMESPACE-security" 

          }
          cleanWs()
        }
   }  
}
