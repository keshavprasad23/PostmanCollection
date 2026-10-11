// Daily PetStore API tests with Newman.
// Put this Jenkinsfile in the same Git repo as:
//   PetStore.postman_collection.json
//   PetStore.postman_environment.json
//   petstore_test_data.csv

pipeline {
    agent any

    // Runs every day at 7 AM IST. "H" picks a fixed minute in that hour so jobs don't all start together.
    triggers {
        cron('''TZ=Asia/Kolkata
H 7 * * *''')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
    }

    environment {
        COLLECTION  = 'PetStore.postman_collection.json'
        ENVIRONMENT = 'PetStore.postman_environment.json'
        DATA_FILE   = 'petstore_test_data.csv'
        REPORT_DIR  = 'newman-reports'
    }

    stages {
        stage('Install Newman') {
            steps {
                sh 'node -v && npm -v'
                sh 'npm install -g newman newman-reporter-htmlextra'
                sh "mkdir -p ${REPORT_DIR}"
            }
        }

        stage('Run API tests: Pet, Store, User') {
            steps {
                sh """
                    newman run ${COLLECTION} -e ${ENVIRONMENT} \
                      --folder "1. Pet" --folder "2. Store" --folder "3. User" \
                      --delay-request 200 \
                      -r cli,junit,htmlextra \
                      --reporter-junit-export ${REPORT_DIR}/functional.xml \
                      --reporter-htmlextra-export ${REPORT_DIR}/functional.html
                """
            }
        }

        stage('Run data-driven tests') {
            steps {
                sh """
                    newman run ${COLLECTION} -e ${ENVIRONMENT} \
                      --folder "4. Data-Driven (use with data file)" \
                      -d ${DATA_FILE} \
                      --delay-request 200 \
                      -r cli,junit,htmlextra \
                      --reporter-junit-export ${REPORT_DIR}/data-driven.xml \
                      --reporter-htmlextra-export ${REPORT_DIR}/data-driven.html
                """
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: "${REPORT_DIR}/*.xml"
            archiveArtifacts artifacts: "${REPORT_DIR}/**", allowEmptyArchive: true
        }
        failure {
            // Needs the Email Extension plugin and SMTP configured in Manage Jenkins > System.
            emailext(
                to: 'keshav.p@droisys.com',
                subject: "PetStore API tests FAILED - build #${env.BUILD_NUMBER}",
                body: "Daily PetStore run failed. Report: ${env.BUILD_URL}artifact/${REPORT_DIR}/",
                attachmentsPattern: "${REPORT_DIR}/*.html"
            )
        }
    }
}
