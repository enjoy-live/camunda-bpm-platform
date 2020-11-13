#!/usr/bin/env groovy

// https://github.com/camunda/jenkins-global-shared-library
@Library('camunda-ci') _

List SUPPORTED_DBS = ['postgresql-96': [
                        'image': 'postgres:9.6.18',
                        'profiles': 'postgresql',
                        'extra': ''],
                     'sqlserver-2017': [
                        'image': 'mcr.microsoft.com/mssql/server:2017-latest',
                        'profiles': 'sqlserver',
                        'extra': '-Ddatabase.name="master" -Ddatabase.username=sa -Ddatabase.password=cambpm-123#']
                    ]

String getAgent(Integer cpuLimit = 4){
  String mavenForkCount = cpuLimit;
  String mavenMemoryLimit = cpuLimit * 2;
  """
metadata:
  labels:
    agent: ci-cambpm-camunda-cloud-build
spec:
  nodeSelector:
    cloud.google.com/gke-nodepool: agents-n1-standard-32-netssd-preempt
  tolerations:
  - key: "agents-n1-standard-32-netssd-preempt"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: "jnlp"
    image: "gcr.io/ci-30-162810/centos:v0.4.6"
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    tty: true
    env:
    - name: LIMITS_CPU
      value: ${mavenForkCount}
    - name: TZ
      value: Europe/Berlin
    resources:
      limits:
        cpu: ${cpuLimit}
        memory: ${mavenMemoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${mavenMemoryLimit}Gi
    workingDir: "/home/work"
    volumeMounts:
      - mountPath: /home/work
        name: workspace-volume
  """
}

String getDbAgent(String database) {
  if (database.contains('postgres')) {
    return getPostgresAgent(SUPPORTED_DBS[database]['image'])
  }

  if (database.contains('sqlserver')) {
    return getSqlServerAgent(SUPPORTED_DBS[database]['image'])
  }
}

String getPostgresAgent(String dockerTag = '9.6.18', Integer cpuLimit = 1){
  // assuming 2Gig for each core
  String memoryLimit = cpuLimit * 2;
  """
  - name: postgres
    image: postgres:${dockerTag}
    env:
    - name: TZ
      value: Europe/Berlin
    - name: POSTGRES_DB
      value: process-engine
    - name: POSTGRES_USER
      value: camunda
    - name: POSTGRES_PASSWORD
      value: camunda
    resources:
      limits:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
  """
}

String getSqlServerAgent(String dockerTag = '2017-latest', Integer cpuLimit = 1){
  String memoryLimit = cpuLimit * 2;
  """
  - name: mcr.microsoft.com/mssql/server
    image: mcr.microsoft.com/mssql/server:${dockerTag}
    env:
    - name: TZ
      value: Europe/Berlin
    - name: ACCEPT_EULA
      value: Y
    - name: SA_PASSWORD
      value: cambpm-123#
    resources:
      limits:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
  """
}

String getChromeAgent(Integer cpuLimit = 1){
  String memoryLimit = cpuLimit * 2;
  """
  - name: chrome
    image: 'gcr.io/ci-30-162810/chrome:78v0.1.2'
    command: ["cat"]
    tty: true
    env:
    - name: TZ
      value: Europe/Berlin
    resources:
      limits:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${memoryLimit}Gi
  """
}

pipeline {
  agent none
  stages {
    stage('ASSEMBLY') {
      agent {
        kubernetes {
          yaml getAgent(16)
        }
      }
      steps {
        withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
          sh '''
            mvn --version
            java -version
          '''
          nodejs('nodejs-14.6.0'){
            sh '''
              node -v
              npm version
            '''
            configFileProvider([configFile(fileId: 'maven-nexus-settings', variable: 'MAVEN_SETTINGS_XML')]) {
              sh """
                node -v
                npm version
                mvn -s \$MAVEN_SETTINGS_XML -T\$LIMITS_CPU clean install source:jar -Pdistro,distro-ce,distro-wildfly,distro-webjar -DskipTests -Dmaven.repo.local=\$(pwd)/.m2 com.mycila:license-maven-plugin:check -B
              """
            }
          }
            stash name: "platform-stash-runtime", includes: ".m2/org/camunda/**/*-SNAPSHOT/**", excludes: "**/qa/**,**/*qa*/**,**/*.zip,**/*.tar.gz"
            stash name: "platform-stash-qa", includes: ".m2/org/camunda/bpm/**/qa/**/*-SNAPSHOT/**,.m2/org/camunda/bpm/**/*qa*/**/*-SNAPSHOT/**", excludes: "**/*.zip,**/*.tar.gz"
            stash name: "platform-stash-distro", includes: ".m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.zip,.m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.tar.gz"
        }
      }
    }
//    stage('h2 tests') {
//      parallel {
//        stage('engine-UNIT-h2') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('h2')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent(16)
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              //runMaven(true, false,'engine/', '-T\$LIMITS_CPU test -Pdatabase,h2')
//            }
//          }
//        }
//        stage('engine-UNIT-authorizations-h2') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('h2')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent(16)
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              //runMaven(true, false,'engine/', '-T\$LIMITS_CPU test -Pdatabase,h2,cfgAuthorizationCheckRevokesAlways')
//            }
//          }
//        }
//        stage('engine-rest-UNIT-jersey-2') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('rest')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'engine-rest/engine-rest/', 'clean install -Pjersey2')
//            }
//          }
//        }
//        stage('engine-rest-UNIT-resteasy3') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('rest')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'engine-rest/engine-rest/', 'clean install -Presteasy3')
//            }
//          }
//        }
//        stage('webapp-UNIT-h2') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('webapps')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'webapps/', 'clean test -Pdatabase,h2 -Dskip.frontend.build=true')
//            }
//          }
//        }
//        stage('engine-IT-tomcat-9-h2') {// TODO change it to `postgresql-96`
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('IT')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                runMaven(true, true, 'qa/', 'clean install -Ptomcat,h2,engine-integration')
//              }
//            }
//          }
//          post {
//            always {
//              junit testResults: '**/target/*-reports/TEST-*.xml', keepLongStdio: true
//            }
//          }
//        }
//        stage('webapp-IT-tomcat-9-h2') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('webapps', 'IT')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent() + getChromeAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                runMaven(true, true,'qa/', 'clean install -Ptomcat,h2,webapps-integration')
//              }
//            }
//          }
//          post {
//            always {
//              junit testResults: '**/target/*-reports/TEST-*.xml', keepLongStdio: true
//            }
//          }
//        }
//        stage('webapp-IT-standalone-wildfly') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('webapps', 'IT')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent() + getChromeAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                runMaven(true, true,'qa/', 'clean install -Pwildfly-vanilla,webapps-integration-sa')
//              }
//            }
//          }
//        }
//        stage('camunda-run-IT') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('IT', 'run', 'spring-boot')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent() + getChromeAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                runMaven(true, true,'distro/run/', 'clean install -Pintegration-test-camunda-run')
//              }
//            }
//          }
//          post {
//            always {
//              junit testResults: '**/target/*-reports/TEST-*.xml', keepLongStdio: true
//            }
//          }
//        }
//        stage('spring-boot-starter-IT') {
//          when {
//            anyOf {
//              branch 'pipeline-master';
//              allOf {
//                changeRequest();
//                expression {
//                  withLabels('IT', 'spring-boot')
//                }
//              }
//            }
//          }
//          agent {
//            kubernetes {
//              yaml getAgent() + getChromeAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
//                runMaven(true, true,'spring-boot-starter/', 'clean install -Pintegration-test-spring-boot-starter')
//              }
//            }
//          }
//          post {
//            always {
//              junit testResults: '**/target/*-reports/TEST-*.xml', keepLongStdio: true
//            }
//          }
//        }
//      }
//    }
//    stage('db tests + CE webapps IT + EE platform') {
//      parallel {
//        stage('engine-api-compatibility') {
//          agent {
//            kubernetes {
//              yaml getAgent(16)
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'engine/', 'clean verify -Pcheck-api-compatibility')
//            }
//          }
//        }
//        stage('engine-UNIT-plugins') {
//          agent {
//            kubernetes {
//              yaml getAgent(16)
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'engine/', 'clean test -Pcheck-plugins')
//            }
//          }
//        }
//        stage('engine-UNIT-database-table-prefix') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'engine/', 'clean test -Pdb-table-prefix')
//            }
//          }
//        }
//        stage('webapp-UNIT-database-table-prefix') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'webapps/', 'clean test -Pdb-table-prefix')
//            }
//          }
//        }
//        stage('engine-UNIT-wls-compatibility') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, false,'.', 'clean verify -Pcheck-engine,wls-compatibility,jersey')
//            }
//          }
//        }
//        stage('IT-wildfly-domain') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, true,'qa/', 'clean install -Pwildfly-domain,h2,engine-integration')
//            }
//          }
//        }
//        stage('IT-wildfly-servlet') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//              runMaven(true, true,'qa/', 'clean install -Pwildfly,wildfly-servlet,h2,engine-integration')
//            }
//          }
//        }
//        stage('EE-platform-DISTRO-dummy') {
//          agent {
//            kubernetes {
//              yaml getAgent()
//            }
//          }
//          steps{
//            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest') {
//            }
//          }
//        }
//      }
//    }
    stage("engine-UNIT DB tests") {
      matrix {
        axes {
          axis {
            name 'DB'
            values 'postgresql-96', 'postgresql-94', 'postgresql-107', 'postgresql-112', 'postgresql-122',
                    'cockroachdb-201',
                    'mariadb-100', 'mariadb-102', 'mariadb-103',
                    'mariadb-galera',
                    'mysql-57',
                    'oracle-11', 'oracle-12', 'oracle-18', 'oracle-19',
                    'sqlserver-2012', 'sqlserver-2016', 'sqlserver-2017', 'sqlserver-2019',
                    'db2-105', 'db2-111'
          }
        }
        agent {
          kubernetes {
            yaml getAgent() + getDbAgent(env.DB)
            label "engine-UNIT-${env.DB}"
          }
        }
        stages {
          stage("engine-UNIT") {
            steps {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
                runMaven(true, false,'engine/', 'clean test -P' + resolveDbProfiles(env.DB) + " " + resolveExtraMvnProperties(env.DB))
              }
            }
          }
        }
      }
    }
//    stage("engine-UNIT-authorizations DB tests") {
//      matrix {
//        axes {
//          axis {
//            name 'DB'
//            values 'postgresql-96', 'postgresql-94', 'postgresql-107', 'postgresql-112', 'postgresql-122',
//                    'cockroachdb-201',
//                    'mariadb-100', 'mariadb-102', 'mariadb-103',
//                    'mariadb-galera',
//                    'mysql-57',
//                    'oracle-11', 'oracle-12', 'oracle-18', 'oracle-19',
//                    'sqlserver-2012', 'sqlserver-2016', 'sqlserver-2017', 'sqlserver-2019',
//                    'db2-105', 'db2-111'
//          }
//        }
//        agent {
//          kubernetes {
//            yaml getAgent() + getDbAgent(env.DB)
//            label "engine-UNIT-authorizations-${env.DB}"
//          }
//        }
//        stages {
//          stage("engine-UNIT-authorizations") {
//            steps {
//              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//                runMaven(true, false,'engine/', 'clean test -PcfgAuthorizationCheckRevokesAlways' + resolveDbProfiles(env.DB))
//              }
//            }
//          }
//        }
//      }
//    }
//    stage("webapp-UNIT DB tests") {
//      matrix {
//        axes {
//          axis {
//            name 'DB'
//            values 'postgresql-96', 'postgresql-94', 'postgresql-107', 'postgresql-112', 'postgresql-122',
//                    'cockroachdb-201',
//                    'mariadb-100', 'mariadb-102', 'mariadb-103',
//                    'mariadb-galera',
//                    'mysql-57',
//                    'oracle-11', 'oracle-12', 'oracle-18', 'oracle-19',
//                    'sqlserver-2012', 'sqlserver-2016', 'sqlserver-2017', 'sqlserver-2019',
//                    'db2-105', 'db2-111'
//          }
//        }
//        agent {
//          kubernetes {
//            yaml getAgent() + getDbAgent(env.DB)
//            label "webapp-UNIT-${env.DB}"
//          }
//        }
//        stages {
//          stage("webapp-UNIT") {
//            steps {
//              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//                runMaven(true, false,'webapps/', 'clean test -Dskip.frontend.build=true -P' + resolveDbProfiles(env.DB))
//              }
//            }
//          }
//        }
//      }
//    }
//    stage("webapp-UNIT-authorizations DB tests") {
//      matrix {
//        axes {
//          axis {
//            name 'DB'
//            values 'postgresql-96', 'postgresql-94', 'postgresql-107', 'postgresql-112', 'postgresql-122',
//                    'cockroachdb-201',
//                    'mariadb-100', 'mariadb-102', 'mariadb-103',
//                    'mariadb-galera',
//                    'mysql-57',
//                    'oracle-11', 'oracle-12', 'oracle-18', 'oracle-19',
//                    'sqlserver-2012', 'sqlserver-2016', 'sqlserver-2017', 'sqlserver-2019',
//                    'db2-105', 'db2-111'
//          }
//        }
//        agent {
//          kubernetes {
//            yaml getAgent() + getDbAgent(env.DB)
//            label "webapp-UNIT-authorizations-${env.DB}"
//          }
//        }
//        stages {
//          stage("webapp-UNIT-authorizations") {
//            steps {
//              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'maven-nexus-settings') {
//                runMaven(true, false,'webapps/', 'clean test -Dskip.frontend.build=true -PcfgAuthorizationCheckRevokesAlways' + resolveDbProfiles(env.DB))
//              }
//            }
//          }
//        }
//      }
//    }
  }
  post {
    changed {
      script {
        if (!agentDisconnected()){ 
          // send email if the slave disconnected
        }
      }
    }
    always {
      script {
        if (agentDisconnected()) {// Retrigger the build if the slave disconnected
          build job: currentBuild.projectName, propagate: false, quietPeriod: 60, wait: false
        }
      }
    }
  }
}

void runMaven(boolean runtimeStash, boolean distroStash, String directory, String cmd) {
  if (runtimeStash) unstash "platform-stash-runtime"
  if (distroStash) unstash "platform-stash-distro"
  configFileProvider([configFile(fileId: 'maven-nexus-settings', variable: 'MAVEN_SETTINGS_XML')]) {
    sh("export MAVEN_OPTS='-Dmaven.repo.local=\$(pwd)/.m2' && cd ${directory} && mvn -s \$MAVEN_SETTINGS_XML ${cmd} -B")
  }
}

void withLabels(String... labels) {
  for ( l in labels) {
    pullRequest.labels.contains(labelName)
  }
}

String resolveDbProfiles(String database) {
  return SUPPORTED_DBS[database]['profiles'];
}

String resolveExtraMvnProperties(String database) {
  return SUPPORTED_DBS[database]['extra'];
}