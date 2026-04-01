pipeline {
    agent any

    environment {
        // Set to 'true' to intentionally fail the pipeline for demo purposes
        FORCE_FAIL = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    script {
                        if (env.FORCE_FAIL == 'true') {
                            error('FORCE_FAIL=true — Build intentionally failed for demo.')
                        }
                    }
                    echo 'Building application... Done.'
                }
            }
        }

        stage('Unit Test') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    echo 'Running unit tests... All passed.'
                }
            }
        }

        stage('Build Docker Image & Push to ECR') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    echo 'Configuring AWS credentials... Done.'
                    echo 'Logging in to ECR... Done.'
                    echo 'Building and pushing Docker image to ECR... Done.'
                }
            }
        }

        stage('Notify DevOps Agent') {
            when {
                expression { currentBuild.currentResult == 'FAILURE' }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_URL',    variable: 'DEVOPS_AGENT_WEBHOOK_URL'),
                    string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_SECRET', variable: 'DEVOPS_AGENT_WEBHOOK_SECRET')
                ]) {
                    sh '''
                        TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
                        INCIDENT_ID=$(cat /proc/sys/kernel/random/uuid)
                        CLEAN_SECRET=$(echo -n "$DEVOPS_AGENT_WEBHOOK_SECRET" | tr -d '[:space:]')

                        PAYLOAD=$(printf \
                            '{"eventType":"incident","incidentId":"%s","action":"created","priority":"HIGH","title":"[JENKINS] actionprofile pipeline failed on %s","description":"Source: Jenkins Pipeline. One or more stages failed in the actionprofile pipeline. Investigate the failure related to the latest commit. Job:%s Branch:%s Commit:%s Build URL:%s 1. Review changes in the latest commit 2. Analyze Jenkins stage logs 3. Identify root cause and suggest a fix","service":"actionprofile-jenkins","timestamp":"%s","data":{"metadata":{"source":"Jenkins","job":"%s","branch":"%s","commit":"%s","build_number":"%s","build_url":"%s"}}}' \
                            "$INCIDENT_ID" \
                            "${BRANCH_NAME:-main}" \
                            "${JOB_NAME}" \
                            "${BRANCH_NAME:-main}" \
                            "${GIT_COMMIT:-unknown}" \
                            "${BUILD_URL}" \
                            "$TIMESTAMP" \
                            "${JOB_NAME}" \
                            "${BRANCH_NAME:-main}" \
                            "${GIT_COMMIT:-unknown}" \
                            "${BUILD_NUMBER}" \
                            "${BUILD_URL}")

                        SIGNATURE=$(echo -n "${TIMESTAMP}:${PAYLOAD}" | \
                            openssl dgst -sha256 -hmac "$CLEAN_SECRET" -binary | \
                            base64 -w 0)

                        HTTP_STATUS=$(curl -s -o /tmp/response.txt -w "%{http_code}" \
                            -X POST "$DEVOPS_AGENT_WEBHOOK_URL" \
                            -H "Content-Type: application/json" \
                            -H "x-amzn-event-timestamp: $TIMESTAMP" \
                            -H "x-amzn-event-signature: $SIGNATURE" \
                            -d "$PAYLOAD")

                        echo "DevOps Agent webhook -> HTTP $HTTP_STATUS"
                        cat /tmp/response.txt
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
