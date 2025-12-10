pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Choose environment'
        )
    }

    environment {
        SONAR_SERVER = "sonarqube"        // Name from Jenkins Global Configuration
        SONAR_TOKEN  = "sonar-token"      // Jenkins Secret Text credential ID
        TOMCAT_CRED  = "local tomcat user"
        WAR_BACKUP_DIR = "backup"
    }

    stages {

        /* ================== SELECT ENVIRONMENT ================== */
        stage('Select Environment') {
            steps {
                script {
                    def BRANCH = (env.BRANCH_NAME ?: "main").toLowerCase()

                    env.DEPLOY_ENV =
                        (BRANCH.startsWith('dev'))  ? 'dev'  :
                        (BRANCH.startsWith('qa'))   ? 'qa'   :
                        (BRANCH.startsWith('prod') || BRANCH == 'main') ? 'prod' :
                                                                           params.DEPLOY_ENV

                    echo "Branch Name: ${BRANCH}"
                    echo "Final Selected Environment = ${env.DEPLOY_ENV}"
                }
            }
        }

        /* ================== SONARQUBE ANALYSIS ================== */
        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv("${SONAR_SERVER}") {
                        withCredentials([string(credentialsId: SONAR_TOKEN, variable: 'TOKEN')]) {

                            sh '''
                                sonar-scanner \
                                    -Dsonar.projectKey=Amazon \
                                    -Dsonar.projectName=Amazon \
                                    -Dsonar.sources=Amazon \
                                    -Dsonar.java.binaries=Amazon/Amazon-Web/target \
                                    -Dsonar.host.url=$SONAR_HOST_URL \
                                    -Dsonar.login=$TOKEN
                            '''
                        }
                    }
                }
            }
        }

        /* ================== COMPILE ================== */
        stage('Compile') {
            steps {
                dir('Amazon') {
                    sh 'mvn clean compile -B'
                }
            }
        }

        /* ================== UNIT TEST ================== */
        stage('Unit Test') {
            steps {
                dir('Amazon') {
                    sh 'mvn test -B'
                }
                junit '*/target/surefire-reports/.xml'
            }
        }

        /* ================== PACKAGE WAR ================== */
        stage('Package WAR') {
            steps {
                dir('Amazon') {
                    sh 'mvn -DskipTests package -B'
                }
            }
        }

        /* ================== BACKUP WAR ================== */
        stage('Backup WAR') {
            steps {
                sh "mkdir -p ${WAR_BACKUP_DIR}"
                sh "cp Amazon/Amazon-Web/target/*.war ${WAR_BACKUP_DIR}/amazon_${BUILD_NUMBER}.war"
                echo "WAR backup saved: ${WAR_BACKUP_DIR}/amazon_${BUILD_NUMBER}.war"
            }
        }

        /* ================== DEPLOY TO TOMCAT ================== */
        stage('Deploy to Tomcat') {
            steps {
                script {
                    def HOST =
                        (env.DEPLOY_ENV == 'dev')  ? "localhost:8085" :
                        (env.DEPLOY_ENV == 'qa')   ? "localhost:8092" :
                                                      "localhost:8081"

                    echo "Deploying to: ${HOST}"

                    withCredentials([usernamePassword(
                        credentialsId: TOMCAT_CRED,
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {

                        sh """
                            echo "===== Undeploying old version ====="
                            curl -s -u $USER:$PASS "http://$HOST/manager/text/undeploy?path=/amazon" || true

                            echo "===== Deploying new WAR ====="
                            curl -s -u $USER:$PASS --upload-file Amazon/Amazon-Web/target/*.war \
                                "http://$HOST/manager/text/deploy?path=/amazon&update=true"

                            echo "===== Deployment Completed ====="
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${WAR_BACKUP_DIR}/*.war", onlyIfSuccessful: false
        }
    }
}
