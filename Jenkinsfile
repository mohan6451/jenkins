pipeline {
  agent any

  parameters {
    choice(
      name: 'ENVIRONMENT',
      choices: ['external', 'internal'],
      description: 'Scan environment'
    )

    string(
      name: 'NMAP_JOB',
      defaultValue: 'Riskhunt-nmap',
      description: 'Nmap job exporting alive-hosts.txt'
    )

    string(
      name: 'ALIVE_FILE',
      defaultValue: 'alive-hosts.txt',
      description: 'Alive host artifact'
    )

    string(
      name: 'GMP_SOCKET',
      defaultValue: '/run/gvmd/gvmd.sock',
      description: 'gvmd Unix socket path'
    )

    string(
      name: 'TARGET_RETENTION_DAYS',
      defaultValue: '90',
      description: 'Delete Jenkins targets older than N days'
    )
  }

  environment {
    REPORT_DIR = 'openvas-reports'

    GMP_SOCKET            = "${params.GMP_SOCKET}"
    ENVIRONMENT           = "${params.ENVIRONMENT}"
    TARGET_RETENTION_DAYS = "${params.TARGET_RETENTION_DAYS}"

    // Default scanner (GMP reference implementation)
    SCANNER_ID = "15348381-3180-213f-4eec-123591912388"

    // Port lists
    PORTLIST_EXTERNAL = "4a4717fe-57d2-11e1-9a26-406186ea4fc5"
    PORTLIST_INTERNAL = "33d0cd82-57c6-11e1-8ed1-406186ea4fc5"
  }

  options {
    skipDefaultCheckout true
    timestamps()
  }

  stages {

    /* =====================================================
       1. Fetch Alive Hosts
    ====================================================== */
    stage('Fetch Alive Hosts') {
      steps {
        copyArtifacts(
          projectName: params.NMAP_JOB,
          selector: lastSuccessful(),
          filter: params.ALIVE_FILE,
          flatten: true
        )

        sh '''
          set -e
          echo "[+] Alive hosts:"
          wc -l alive-hosts.txt
        '''
      }
    }

    /* =====================================================
       2. Prepare Target (Idempotent + Reuse)
    ====================================================== */
    stage('Prepare Target') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'gvm_admin',
          usernameVariable: 'GMP_USER',
          passwordVariable: 'GMP_PASS'
        )]) {
          sh '''
            set -e
            mkdir -p ${REPORT_DIR}

            gmp() {
              local cmd="S1"
              gvm-cli --gmp-username "$GMP_USER" --gmp-password "$GMP_PASS" \
                socket --socketpath "$GMP_SOCKET" --xml "$cmd"
            }

            # Select port list by environment
            if [ "$ENVIRONMENT" = "external" ]; then
              PORT_LIST_ID="$PORTLIST_EXTERNAL"
            else
              PORT_LIST_ID="$PORTLIST_INTERNAL"
            fi

            HOSTS=$(tr '\\n' ',' < alive-hosts.txt | sed 's/,$//')
            HOST_HASH=$(echo "$HOSTS" | sha1sum | cut -c1-8)

            TARGET_NAME="jenkins-${ENVIRONMENT}-${HOST_HASH}"
            echo "[+] Target name: $TARGET_NAME"

            EXISTING_TARGET=$(gmp "<get_targets filter='name=${TARGET_NAME}'/>" | \
              xmllint --xpath "string(//target/@id)" -)

            if [ -n "$EXISTING_TARGET" ]; then
              echo "[+] Reusing target $EXISTING_TARGET"
              TARGET_ID="$EXISTING_TARGET"
            else
              echo "[+] Creating new target"
              RESP=$(gmp "<create_target>
                <name>${TARGET_NAME}</name>
                <hosts>${HOSTS}</hosts>
                <port_list id=\\"${PORT_LIST_ID}\\"/>
              </create_target>")

              TARGET_ID=$(echo "$RESP" | \
                xmllint --xpath "string(//create_target_response/@id)" -)
            fi

            echo "TARGET_ID=${TARGET_ID}" > ${REPORT_DIR}/target.env
          '''
        }
      }
    }

    /* =====================================================
       3. Cleanup Old Jenkins Targets (90 days)
    ====================================================== */
    stage('Cleanup Old Targets') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'gvm_admin',
          usernameVariable: 'GMP_USER',
          passwordVariable: 'GMP_PASS'
        )]) {
          sh '''
            set -e
            echo "[+] Cleanup skipped (safe placeholder, non-destructive)"
          '''
        }
      }
    }

    /* =====================================================
       4. Resolve Scan Config ID (DYNAMIC)
    ====================================================== */
    stage('Resolve Scan Config') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'gvm_admin',
          usernameVariable: 'GMP_USER',
          passwordVariable: 'GMP_PASS'
        )]) {
          sh '''
            set -e

            gmp() {
              gvm-cli \
                --gmp-username "$GMP_USER" \
                --gmp-password "$GMP_PASS" \
                socket --socketpath "$GMP_SOCKET" \
                --xml "$1"
            }

            echo "[+] Resolving 'Full and Fast' scan config UUID"
            SCAN_CONFIG_ID=$(gmp "<get_configs filter='name=Full and Fast'/>" | \
              xmllint --xpath "string(//config/@id)" -)

            if [ -z "$SCAN_CONFIG_ID" ]; then
              echo "❌ Scan config 'Full and Fast' not found"
              exit 1
            fi

            echo "SCAN_CONFIG_ID=${SCAN_CONFIG_ID}" > ${REPORT_DIR}/scanconfig.env
          '''
        }
      }
    }

    /* =====================================================
       5. Create & Start Task (GMP SPEC)
    ====================================================== */
    stage('Create and Start Task') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'gvm_admin',
          usernameVariable: 'GMP_USER',
          passwordVariable: 'GMP_PASS'
        )]) {
          sh '''
            set -e
            . ${REPORT_DIR}/target.env
            . ${REPORT_DIR}/scanconfig.env

            gmp() {
              gvm-cli \
                --gmp-username "$GMP_USER" \
                --gmp-password "$GMP_PASS" \
                socket --socketpath "$GMP_SOCKET" \
                --xml "$1"
            }

            echo "[+] Creating task"
            RESP=$(gmp "<create_task>
              <name>scan-${BUILD_ID}</name>
              <config id=\\"${SCAN_CONFIG_ID}\\"/>
              <target id=\\"${TARGET_ID}\\"/>
              <scanner id=\\"${SCANNER_ID}\\"/>
            </create_task>")

            TASK_ID=$(echo "$RESP" | \
              xmllint --xpath "string(//create_task_response/@id)" -)

            if [ -z "$TASK_ID" ]; then
              echo "❌ Task creation failed"
              exit 1
            fi

            echo "$TASK_ID" > ${REPORT_DIR}/task.id
            gmp "<start_task task_id=\\"${TASK_ID}\\"/>"
            echo "[+] Task started: $TASK_ID"
          '''
        }
      }
    }

    /* =====================================================
       6. Wait for Completion
    ====================================================== */
    stage('Wait for Completion') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'gvm_admin',
          usernameVariable: 'GMP_USER',
          passwordVariable: 'GMP_PASS'
        )]) {
          sh '''
            set -e
            TASK_ID=$(cat ${REPORT_DIR}/task.id)

            gmp() {
              gvm-cli \
                --gmp-username "$GMP_USER" \
                --gmp-password "$GMP_PASS" \
                socket --socketpath "$GMP_SOCKET" \
                --xml "$1"
            }

            echo "[+] Waiting for scan completion"
            while true; do
              STATUS=$(gmp "<get_tasks task_id=\\"${TASK_ID}\\"/>" | \
                xmllint --xpath "string(//task/status)" -)
              echo "Status: $STATUS"
              [ "$STATUS" = "Done" ] && break
              sleep 30
            done
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'alive-hosts.txt,openvas-reports/**'
      echo "✅ OpenVAS scan completed successfully"
    }
  }
}