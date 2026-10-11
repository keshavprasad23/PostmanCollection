// Daily PetStore API tests with Newman, for a Jenkins server running on WINDOWS.
// Commit this file to the root of the GitHub repo, next to:
//   PetStore.postman_collection.json
//   PetStore.postman_environment.json
//   petstore_test_data.json

pipeline {
    agent any

    // Runs every day at 7 AM India time. "H" picks a fixed minute in that hour.
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
        DATA_FILE   = 'petstore_test_data.json'
        REPORT_DIR  = 'newman-reports'
    }

    stages {
        stage('Install Newman') {
            steps {
                // Installs Newman inside the job workspace, so it works whichever
                // Windows account the Jenkins service runs as.
                bat 'node -v && npm -v'
                bat 'npm install --no-save --no-audit --no-fund newman newman-reporter-htmlextra'
                bat "if not exist ${REPORT_DIR} mkdir ${REPORT_DIR}"
            }
        }

        stage('Run API tests: Pet, Store, User') {
            steps {
                bat """
                    npx newman run ${COLLECTION} -e ${ENVIRONMENT} ^
                      --folder "1. Pet" --folder "2. Store" --folder "3. User" ^
                      --delay-request 200 --color off ^
                      -r cli,junit,htmlextra ^
                      --reporter-junit-export ${REPORT_DIR}\\functional.xml ^
                      --reporter-htmlextra-export ${REPORT_DIR}\\functional.html
                """
            }
        }

        stage('Run data-driven tests') {
            steps {
                bat """
                    npx newman run ${COLLECTION} -e ${ENVIRONMENT} ^
                      --folder "4. Data-Driven (use with data file)" ^
                      -d ${DATA_FILE} ^
                      --delay-request 200 --color off ^
                      -r cli,junit,htmlextra ^
                      --reporter-junit-export ${REPORT_DIR}\\data-driven.xml ^
                      --reporter-htmlextra-export ${REPORT_DIR}\\data-driven.html
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
            // Needs the Email Extension plugin and SMTP set up (see the setup guide, step 5).
            emailext(
                to: 'keshav.p@droisys.com',
                subject: "PetStore API tests FAILED - build #${env.BUILD_NUMBER}",
                body: "The daily PetStore run failed.\nResults: ${env.BUILD_URL}testReport/\nReports: ${env.BUILD_URL}artifact/${REPORT_DIR}/",
                attachmentsPattern: "${REPORT_DIR}/*.html"
            )
        }
    }
}
