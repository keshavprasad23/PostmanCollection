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

    parameters {
        booleanParam(name: 'VERBOSE_LOG', defaultValue: false,
            description: 'Tick to print every passing assertion in the console (default: only failures).')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
    }

    environment {
        COLLECTION   = 'PetStore.postman_collection.json'
        ENVIRONMENT  = 'PetStore.postman_environment.json'
        DATA_FILE    = 'petstore_test_data.json'
        REPORT_DIR   = 'newman-reports'
        NODE_OPTIONS = '--no-deprecation'      // hides Node.js deprecation warnings in the log
    }

    stages {
        stage('Setup') {
            steps {
                echo '>>> Installing Newman and preparing the report folder'
                bat '@node -v && npm -v'
                bat '@npm install --no-save --no-audit --no-fund --loglevel=error newman newman-reporter-htmlextra'
                bat "@if exist ${REPORT_DIR} rmdir /s /q ${REPORT_DIR}"
                bat "@mkdir ${REPORT_DIR}"
                writeFile file: 'ci-summary.js', text: summaryJs()
            }
        }

        stage('Functional tests') {
            steps {
                echo '>>> Running Pet, Store and User tests'
                // catchError: if tests fail, mark the build FAILED but still run the next suite.
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    bat newmanCmd('--folder "1. Pet" --folder "2. Store" --folder "3. User"', 'functional')
                }
            }
        }

        stage('Data-driven tests') {
            steps {
                echo ">>> Running data-driven tests with ${DATA_FILE}"
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    bat newmanCmd("--folder \"4. Data-Driven (use with data file)\" -d ${DATA_FILE}", 'data-driven')
                }
            }
        }
    }

    post {
        // Runs after EVERY build: summary in the console, results in Jenkins, email to the team.
        // Email needs the Email Extension plugin and SMTP set up (Manage Jenkins > System > Extended E-mail Notification).
        always {
            script {
                bat(script: "@node ci-summary.js ${REPORT_DIR}", returnStatus: true)   // never blocks the email

                def t = junit allowEmptyResults: true, testResults: "${REPORT_DIR}/*.xml"
                archiveArtifacts artifacts: "${REPORT_DIR}/*.html, ${REPORT_DIR}/*.xml", allowEmptyArchive: true

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

// Builds the Newman command for one suite.
def newmanCmd(String selection, String name) {
    def quiet = params.VERBOSE_LOG ? '' : '--reporter-cli-no-success-assertions'
    return "@npx newman run ${env.COLLECTION} -e ${env.ENVIRONMENT} ${selection} " +
           "--delay-request 200 --color off --reporter-cli-no-banner ${quiet} " +
           "-r cli,junit,htmlextra,json " +
           "--reporter-junit-export ${env.REPORT_DIR}\\${name}.xml " +
           "--reporter-htmlextra-export ${env.REPORT_DIR}\\${name}.html " +
           "--reporter-json-export ${env.REPORT_DIR}\\${name}.json"
}

// Node.js script that prints the end-of-build summary table (written to the workspace by the Setup stage).
def summaryJs() {
    return '''// Prints a readable summary of the Newman JSON reports to the Jenkins console.
const fs = require('fs'), path = require('path');
const dir = process.argv[2] || 'newman-reports';
const suites = [
  { file: 'functional.json',  label: 'Functional (Pet / Store / User)' },
  { file: 'data-driven.json', label: 'Data-driven (test data file)', rows: true }
];
const W = 92, line = c => c.repeat(W);
const pad = (s, n) => String(s).padEnd(n).slice(0, n);
const lpad = (s, n) => String(s).padStart(n);
const secs = ms => (ms / 1000).toFixed(1) + 's';

function folderMap(coll) {
  const map = {};
  (function walk(items, folder) {
    (items || []).forEach(it => it.item ? walk(it.item, it.name) : (map[it.id] = folder || '-'));
  })(coll && coll.item, null);
  return map;
}

let grand = { req: 0, tests: 0, failed: 0, ms: 0 }, allFailures = [], allExec = [], rows = [], folders = {};
for (const s of suites) {
  const f = path.join(dir, s.file);
  if (!fs.existsSync(f)) { rows.push({ label: s.label, missing: true }); continue; }
  const run = JSON.parse(fs.readFileSync(f, 'utf8')).run;
  const fmap = folderMap(JSON.parse(fs.readFileSync(f, 'utf8')).collection);
  const st = run.stats, t = run.timings || {};
  const tests = st.assertions.total, failed = st.assertions.failed, req = st.requests.total;
  const dur = (t.completed || 0) - (t.started || 0);
  rows.push({ label: s.label, req, tests, failed, passed: tests - failed, avg: Math.round(t.responseAverage || 0), dur });
  grand.req += req; grand.tests += tests; grand.failed += failed; grand.ms += dur;
  (run.executions || []).forEach(e => {
    const folder = fmap[e.item && e.item.id] || '-';
    const as = e.assertions || [], bad = as.filter(a => a.error).length;
    const fo = folders[folder] = folders[folder] || { req: 0, tests: 0, failed: 0 };
    fo.req++; fo.tests += as.length; fo.failed += bad;
    if (e.requestError) { fo.failed++; }
    allExec.push({ name: e.item.name, method: e.request && e.request.method,
      code: e.response ? e.response.code : 'ERR', ms: e.response ? e.response.responseTime : 0,
      iter: e.cursor ? e.cursor.iteration + 1 : 1, rows: s.rows });
  });
  (run.failures || []).forEach(x => allFailures.push({ suite: s.label,
    request: x.source && x.source.name, test: x.error && (x.error.test || x.error.name),
    msg: x.error && x.error.message, iter: x.cursor ? x.cursor.iteration + 1 : 1, rows: s.rows }));
}

const ok = grand.failed === 0 && allFailures.length === 0 && !rows.some(r => r.missing);
const out = [];
out.push('', line('='));
out.push('  PETSTORE API TEST SUMMARY' + lpad(`Build #${process.env.BUILD_NUMBER || '-'}  |  ${new Date().toLocaleString('en-IN', { timeZone: 'Asia/Kolkata' })} IST`, W - 27));
out.push(line('='));
out.push('  RESULT: ' + (ok ? '[ PASSED ]  All tests passed' : `[ FAILED ]  ${grand.failed} of ${grand.tests} tests failed`));
out.push(line('-'));
out.push('  ' + pad('Suite', 34) + lpad('Requests', 9) + lpad('Tests', 8) + lpad('Passed', 8) + lpad('Failed', 8) + lpad('Avg resp', 10) + lpad('Duration', 10));
out.push('  ' + line('-').slice(0, W - 4));
rows.forEach(r => out.push('  ' + pad(r.label, 34) + (r.missing ? '   (no results - suite did not run)' :
  lpad(r.req, 9) + lpad(r.tests, 8) + lpad(r.passed, 8) + lpad(r.failed, 8) + lpad(r.avg + 'ms', 10) + lpad(secs(r.dur), 10))));
out.push('  ' + line('-').slice(0, W - 4));
out.push('  ' + pad('TOTAL', 34) + lpad(grand.req, 9) + lpad(grand.tests, 8) + lpad(grand.tests - grand.failed, 8) + lpad(grand.failed, 8) + lpad('', 10) + lpad(secs(grand.ms), 10));

out.push('', '  BY FOLDER');
out.push('  ' + pad('Folder', 40) + lpad('Requests', 9) + lpad('Tests', 8) + lpad('Failed', 8) + '   Status');
Object.entries(folders).forEach(([k, v]) => out.push('  ' + pad(k, 40) + lpad(v.req, 9) + lpad(v.tests, 8) + lpad(v.failed, 8) + '   ' + (v.failed ? 'FAIL' : 'PASS')));

const slow = allExec.slice().sort((a, b) => b.ms - a.ms).slice(0, 5);
out.push('', '  SLOWEST REQUESTS');
slow.forEach(e => out.push('  ' + lpad(e.ms + 'ms', 8) + '  ' + pad(e.method, 7) + pad(e.name + (e.rows ? ` (row ${e.iter})` : ''), 50) + '  HTTP ' + e.code));

if (allFailures.length) {
  out.push('', '  FAILED TESTS');
  allFailures.forEach((x, i) => {
    out.push(`  ${i + 1}. ${x.request}${x.rows ? ` (data row ${x.iter})` : ''}  ->  ${x.test}`);
    out.push(`     ${x.msg}`);
  });
}
if (process.env.BUILD_URL) {
  out.push('', '  REPORTS');
  out.push('  Test results : ' + process.env.BUILD_URL + 'testReport/');
  out.push('  HTML reports : ' + process.env.BUILD_URL + 'artifact/' + dir + '/');
}
out.push(line('='), '');
console.log(out.join('\\n'));
'''
}
