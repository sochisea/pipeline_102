// Level 3: Matrix pipeline — run Build/Test/Package across multiple axes
// Works with Jenkinsfile Runner (no plugin-only steps).

pipeline {
  agent any

  options {
    skipDefaultCheckout(true)                            // GitHub Actions already checks out
    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') // normalize UNSTABLE to green
  }

  parameters {
    string(name: 'NAME', defaultValue: 'World', description: 'Who to greet')
    booleanParam(name: 'DO_BUILD', defaultValue: true, description: 'Run Build stage?')
  }

  environment {
    BUILD_ROOT   = 'build'
    REPORTS_ROOT = 'reports/junit'
    DIST_ROOT    = 'dist'
  }

  stages {
    stage('Prepare') {
      steps {
        echo "Hello ${params.NAME}! Preparing workspace…"
        sh '''
          set -eu
          mkdir -p "${BUILD_ROOT}" "${REPORTS_ROOT}" "${DIST_ROOT}"
        '''
      }
    }

    // ─────────────────────────────────────────────────────────────────────────
    // MATRIX STAGE: run the flow across multiple APP_ENV × RUNTIME versions
    // ─────────────────────────────────────────────────────────────────────────
    stage('Matrix: Build • Test • Package') {
      matrix {
        axes {
          axis {
            name 'APP_ENV'       // environment axis (dev/staging)
            values 'dev', 'staging'
          }
          axis {
            name 'RUNTIME'       // version axis (simulate runtimes)
            values 'node18', 'node20'
          }
        }

        // Example of excluding a combination (optional)
        excludes {
          exclude {
            axis { name 'APP_ENV'; value 'dev' }
            axis { name 'RUNTIME'; value 'node20' }
          }
        }

        // Each combination gets its own subfolders to avoid clashes
        stages {
          stage('Build') {
            when { expression { return params.DO_BUILD && env.APP_ENV != 'dev' } } // skip Build on dev
            steps {
              sh '''
                set -eu
                COMBO_BUILD="${BUILD_ROOT}/${APP_ENV}/${RUNTIME}"
                mkdir -p "${COMBO_BUILD}"
                echo "Compiling for ${APP_ENV} on ${RUNTIME}…" > "${COMBO_BUILD}/build.log"
                echo "binary-${APP_ENV}-${RUNTIME}" > "${COMBO_BUILD}/app.bin"
              '''
            }
          }

          stage('Test') {
            steps {
              sh '''
                set -eu
                COMBO_REPORTS="${REPORTS_ROOT}/${APP_ENV}/${RUNTIME}"
                mkdir -p "${COMBO_REPORTS}"
                # Fake JUnit so Jenkins can aggregate results later
                cat > "${COMBO_REPORTS}/tests.xml" <<'XML'
                <?xml version="1.0" encoding="UTF-8"?>
                <testsuite name="demo" tests="2" failures="0" errors="0" skipped="0" time="0.02">
                  <testcase classname="calc.Add" name="adds" time="0.001"/>
                  <testcase classname="calc.Mul" name="multiplies" time="0.001"/>
                </testsuite>
                XML
              '''
            }
          }

          stage('Package') {
            steps {
              sh '''
                set -eu
                COMBO_BUILD="${BUILD_ROOT}/${APP_ENV}/${RUNTIME}"
                COMBO_DIST="${DIST_ROOT}/${APP_ENV}"
                mkdir -p "${COMBO_DIST}"
                # Bundle per-combo outputs (works even if Build was skipped on dev)
                tar -czf "${COMBO_DIST}/app-${APP_ENV}-${RUNTIME}.tar.gz" \
                  -C "${COMBO_BUILD}" . 2>/dev/null || true
                echo "Created package for ${APP_ENV}/${RUNTIME}"
              '''
            }
          }
        } // stages (inside matrix)
      }   // matrix
    }     // stage

    // Aggregate test results from all matrix branches
    stage('Aggregate Reports') {
      steps {
        junit testResults: "${REPORTS_ROOT}/**/*.xml", allowEmptyResults: true
        sh 'echo "All JUnit results aggregated."'
      }
    }
  } // stages

  post {
    always {
      echo 'Level 3 finished. Check artifacts (dist/*) and reports (reports/junit/**/*).'
    }
  }
}
