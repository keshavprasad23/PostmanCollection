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
        // Sends an email after EVERY run - passed or failed.
        // Needs the Email Extension plugin and SMTP set up (Manage Jenkins > System > Extended E-mail Notification).
        always {
            script {
                def t = junit allowEmptyResults: true, testResults: "${REPORT_DIR}/*.xml"
                archiveArtifacts artifacts: "${REPORT_DIR}/**", allowEmptyArchive: true

                def total   = t ? t.totalCount : 0
                def failed  = t ? t.failCount  : 0
                def skipped = t ? t.skipCount  : 0
                def passed  = total - failed - skipped
                def result  = currentBuild.currentResult            // SUCCESS, UNSTABLE or FAILURE
                def ok      = (result == 'SUCCESS')
                def label   = ok ? 'PASSED' : 'FAILED'
                def colour  = ok ? '#1a7f37' : '#cf222e'

                emailext(
                    to: 'keshavprasad23@gmail.com, keshav.p@droisys.com',
                    subject: "${label}: PetStore API tests - build #${env.BUILD_NUMBER} (${passed}/${total} passed)",
                    mimeType: 'text/html',
                    attachmentsPattern: "${REPORT_DIR}/*.html",
                    body: """
                        <div style="font-family:Arial,sans-serif;font-size:14px">
                          <h2 style="color:${colour};margin:0 0 12px">PetStore API tests ${label}</h2>
                          <table cellpadding="6" style="border-collapse:collapse;border:1px solid #ddd">
                            <tr><td><b>Job</b></td><td>${env.JOB_NAME}</td></tr>
                            <tr><td><b>Build</b></td><td>#${env.BUILD_NUMBER}</td></tr>
                            <tr><td><b>Result</b></td><td style="color:${colour}"><b>${result}</b></td></tr>
                            <tr><td><b>Tests passed</b></td><td>${passed} of ${total}</td></tr>
                            <tr><td><b>Tests failed</b></td><td>${failed}</td></tr>
                            <tr><td><b>Duration</b></td><td>${currentBuild.durationString.replace(' and counting', '')}</td></tr>
                          </table>
                          <p>
                            <a href="${env.BUILD_URL}testReport/">Test results</a> &middot;
                            <a href="${env.BUILD_URL}artifact/${REPORT_DIR}/">HTML reports</a> &middot;
                            <a href="${env.BUILD_URL}console">Console log</a>
                          </p>
                          <p style="color:#666">The Newman HTML reports are attached.</p>
                        </div>
                    """
                )
            }
        }
    }
}
