// https://github.com/camunda/jenkins-global-shared-library
// https://github.com/camunda/cambpm-jenkins-shared-library
@Library(['camunda-ci', 'cambpm-jenkins-shared-library@record-only-failing-tests']) _

def failedStageTypes = []

pipeline {
  agent {
    node {
      label 'jenkins-job-runner'
    }
  }
  environment {
    CAMBPM_LOGGER_LOG_LEVEL = 'DEBUG'
  }
  parameters {
    string name: 'EE_DOWNSTREAM', defaultValue: 'cambpm-ee-main-pr/' + cambpmDefaultBranch(), description: 'The name of the EE branch/PR to run the EE pipeline on, e.g. cambpm-ee-main/PR-333'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    copyArtifactPermission('*')
    disableConcurrentBuilds() // TODO https://jira.camunda.com/browse/CAM-13403
  }
  stages {
    stage('ASSEMBLY') {
      when {
        expression {
          env.BRANCH_NAME == cambpmDefaultBranch() || (changeRequest() && !pullRequest.labels.contains('no-build'))
        }
      }
      steps {
        cambpmConditionalRetry([
          agentLabel: 'h2_perf32',
          suppressErrors: false,
          runSteps: {
            cambpmRunMaven('.',
                'clean source:jar deploy source:test-jar com.mycila:license-maven-plugin:check -Pdistro,distro-ce,distro-wildfly,distro-webjar -DaltStagingDirectory=${WORKSPACE}/staging -DskipRemoteStaging=true -DskipTests',
                withCatch: false,
                withNpm: true)

            // archive all .jar, .pom, .xml, .txt runtime artifacts + required .war/.zip/.tar.gz for EE pipeline
            // add a new line for each group of artifacts
            cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/*.jar,.m2/org/camunda/**/*-SNAPSHOT/**/*.pom,.m2/org/camunda/**/*-SNAPSHOT/**/*.xml,.m2/org/camunda/**/*-SNAPSHOT/**/*.txt',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-webapp*frontend-sources.zip',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/license-book*.zip',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-jboss-modules*.zip',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-*-assembly*.tar.gz',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-webapp*.war',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-engine-rest*.war',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-example-invoice*.war',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-h2-webapp*.war',
                                  '.m2/org/camunda/**/*-SNAPSHOT/**/camunda-bpm-run-modules-swaggerui-*-run-swaggerui-license-book-json.json')

            cambpmStash("platform-stash-runtime",
                        ".m2/org/camunda/**/*-SNAPSHOT/**",
                        "**/qa/**,**/*qa*/**,**/*.zip,**/*.tar.gz")
            cambpmStash("platform-stash-archives",
                        ".m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.zip,.m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.tar.gz")
            cambpmStash("platform-stash-qa",
                      ".m2/org/camunda/bpm/**/qa/**/*-SNAPSHOT/**,.m2/org/camunda/bpm/**/*qa*/**/*-SNAPSHOT/**",
                      "**/*.zip,**/*.tar.gz")

            script {
              if (env.BRANCH_NAME == cambpmDefaultBranch()) {
                // CE master triggers EE master
                // otherwise CE PR branch triggers EE PR branch
                eeMainProjectBranch = "cambpm-ee-main/" + cambpmDefaultBranch()
              } else {
                eeMainProjectBranch = params.EE_DOWNSTREAM
              }

              // JOB_NAME, e.g.: '7.15/cambpm-ce/cambpm-main/PR-1373'
              platformVersion = env.JOB_NAME.split('/')[0]

              if (cambpmWithLabels('webapp-integration', 'all-as', 'h2', 'websphere', 'weblogic', 'jbosseap', 'run', 'spring-boot', 'authorizations', 'e2e')) {
                cambpmTriggerDownstream(
                  platformVersion + "/cambpm-ee/" + eeMainProjectBranch,
                  [string(name: 'UPSTREAM_PROJECT_NAME', value: env.JOB_NAME),
                  string(name: 'UPSTREAM_BUILD_NUMBER', value: env.BUILD_NUMBER)],
                  true, true, true
                )
              }

              // TODO: https://jira.camunda.com/browse/CAM-13409
              // only execute on `master`
              // if (cambpmWithLabels()) {
              //   cambpmRunMaven('.',
              //       'org.sonatype.plugins:nexus-staging-maven-plugin:deploy-staged -DaltStagingDirectory=${WORKSPACE}/staging -DskipStaging=true',
              //       withCatch: false,
              //       withNpm: true)
              // }
            }
          },
          postFailure: {
            cambpmPublishTestResult()
          }
        ])

      }
    }
    stage('DB UNIT tests') {
      steps {
        script {
          if(cambpmWithLabels('eng-web', 'h2', 'rolling-update', 'migration', 'default-build', 'authorizations', 'all-db', 'db2', 'mysql', 'oracle', 'mariadb', 'sqlserver', 'postgresql')) {
            cambpmTriggerDownstream(
              platformVersion + "/cambpm-ce/db-unit/${env.BRANCH_NAME}",
              [string(name: 'UPSTREAM_PROJECT_NAME', value: env.JOB_NAME),
              string(name: 'UPSTREAM_BUILD_NUMBER', value: env.BUILD_NUMBER)]
            )
          }

          if(cambpmWithLabels('eng-web', 'h2', 'authorizations')) {
            cambpmTriggerDownstream(
              platformVersion + "/cambpm-ce/db-unit-auth/${env.BRANCH_NAME}",
              [string(name: 'UPSTREAM_PROJECT_NAME', value: env.JOB_NAME),
              string(name: 'UPSTREAM_BUILD_NUMBER', value: env.BUILD_NUMBER)]
            )
          }
        }
      }
    }
    stage('UNIT history level, engine IT, webapp IT') {
      parallel {
        stage('engine-UNIT-historylevel-none') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'verify -Pcfghistorynone', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-UNIT-historylevel-audit') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'verify -Pcfghistoryaudit', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              },
            ])
          }
        }
        stage('engine-UNIT-historylevel-activity') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'verify -Pcfghistoryactivity', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-IT-tomcat-9-postgresql-96') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'postgresql_96',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Ptomcat,postgresql,engine-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-IT-wildfly-postgresql-96') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'postgresql_96',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly,postgresql,engine-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
                cambpmAddFailedStageType(failedStageTypes, 'engine-IT-wildfly')
                cambpmArchiveArtifacts('qa/wildfly-runtime/target/**/standalone/log/server.log')
              }
            ])
          }
        }
        stage('engine-IT-XA-wildfly-postgresql-96') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'postgresql_96',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly,postgresql,postgresql-xa,engine-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
                cambpmArchiveArtifacts('qa/wildfly-runtime/target/**/standalone/log/server.log')
              }
            ])
          }
        }
        stage('webapp-IT-tomcat-9-h2') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Ptomcat,h2,webapps-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('webapp-IT-wildfly-h2') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly,h2,webapps-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('webapp-IT-standalone-tomcat-9') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Ptomcat-vanilla,webapps-integration-sa', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('webapp-IT-standalone-wildfly') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly-vanilla,webapps-integration-sa', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('camunda-run-IT') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('distro/run/', 'clean install -Pintegration-test-camunda-run', runtimeStash: true, archiveStash: true, qaStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('spring-boot-starter-IT') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'chrome_78',
              runSteps: {
                cambpmRunMaven('spring-boot-starter/', 'clean install -Pintegration-test-spring-boot-starter', runtimeStash: true, archiveStash: true, qaStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
      }
    }
    stage('Engine Rest UNIT tests') {
      steps {
        script {
          // see the .ci/config/matrices.yaml for the stage generation values
          // see .ci/config/stage-types.yaml for the stage configurations
          parallel(cambpmGetMatrixStages('engine-rest', failedStageTypes, { stageInfo ->
            return cambpmWithLabels(stageInfo.allowedLabels)
          }))
        }
      }
    }
    stage('MISC tests') {
      parallel {
        stage('engine-api-compatibility') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'clean verify -Pcheck-api-compatibility', runtimeStash: true)
              }
            ])
          }
        }
        stage('engine-UNIT-plugins') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'clean test -Pcheck-plugins', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-UNIT-database-table-prefix') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('engine/', 'clean test -Pdb-table-prefix', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('webapp-UNIT-database-table-prefix') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('webapps/', 'clean test -Pdb-table-prefix -Dskip.frontend.build=true', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-UNIT-wls-compatibility') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('.', 'clean verify -Pcheck-engine,wls-compatibility,jersey', runtimeStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-IT-wildfly-domain') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly-domain,h2,engine-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
        stage('engine-IT-wildfly-servlet') {
          steps {
            cambpmConditionalRetry([
              agentLabel: 'h2',
              runSteps: {
                cambpmRunMaven('qa/', 'clean install -Pwildfly,wildfly-servlet,h2,engine-integration', runtimeStash: true, archiveStash: true)
              },
              postFailure: {
                cambpmPublishTestResult()
              }
            ])
          }
        }
      }
    }
  }
  post {
    changed {
      script {
        if (!agentDisconnected()){
          cambpmSendEmailNotification()
        }
      }
    }
  }
}
