// ============================================================
// Jenkinsfile — sync-service
// Declarative Pipeline
// Environments: qa | staging | prod
// ============================================================

pipeline {
    agent { label 'gcp-builder' }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    // ── Derived from branch / tag ────────────────────────────
    environment {
        APP_NAME        = 'sync-service'
        GCP_PROJECT     = credentials('gcp-project-id')
        GAR_REGION      = 'asia-south1'
        GAR_REPO        = 'cloudeagle-docker'
        IMAGE_BASE      = "${GAR_REGION}-docker.pkg.dev/${GCP_PROJECT}/${GAR_REPO}/${APP_NAME}"
        IMAGE_TAG       = "${GIT_COMMIT[0..7]}"
        IMAGE_FULL      = "${IMAGE_BASE}:${IMAGE_TAG}"
        // Resolved per-branch below in stages
        DEPLOY_ENV      = resolveEnv()
        LAST_STABLE_TAG = readLastStableTag()
    }

    stages {

        // ── 1. CHECKOUT ─────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_AUTHOR = sh(script: "git log -1 --format='%an'", returnStdout: true).trim()
                    echo "Branch: ${BRANCH_NAME} | Author: ${env.GIT_AUTHOR} | Commit: ${IMAGE_TAG}"
                }
            }
        }

        // ── 2. TESTS ─────────────────────────────────────────
        stage('Unit Tests') {
            steps {
                sh './mvnw test -Dspring.profiles.active=test'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                    jacoco execPattern: 'target/jacoco.exec', minimumInstructionCoverage: '70'
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './mvnw verify -Pfailsafe -Dspring.profiles.active=test'
            }
            post {
                always {
                    junit 'target/failsafe-reports/**/*.xml'
                }
            }
        }

        // ── 3. STATIC ANALYSIS ───────────────────────────────
        stage('SAST & Code Quality') {
            parallel {
                stage('SpotBugs') {
                    steps {
                        sh './mvnw spotbugs:check'
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        sh './mvnw org.owasp:dependency-check-maven:check'
                        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                    }
                }
                stage('SonarQube') {
                    when { not { changeRequest() } }   // skip on PRs if Sonar is slow
                    steps {
                        withSonarQubeEnv('sonarqube-server') {
                            sh './mvnw sonar:sonar'
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
        }

        // ── 4. BUILD JAR ─────────────────────────────────────
        stage('Build JAR') {
            steps {
                sh './mvnw package -DskipTests -Dspring.profiles.active=${DEPLOY_ENV}'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // ── 5. DOCKER BUILD & PUSH ───────────────────────────
        stage('Docker Build & Push') {
            when { not { changeRequest() } }   // PRs do not push images
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh """
                        gcloud auth activate-service-account --key-file=\${GOOGLE_APPLICATION_CREDENTIALS}
                        gcloud auth configure-docker ${GAR_REGION}-docker.pkg.dev --quiet
                        docker build \\
                            --build-arg BUILD_ENV=${DEPLOY_ENV} \\
                            --label git.commit=${IMAGE_TAG} \\
                            --label build.number=${BUILD_NUMBER} \\
                            -t ${IMAGE_FULL} \\
                            -t ${IMAGE_BASE}:${DEPLOY_ENV}-latest \\
                            .
                        docker push ${IMAGE_FULL}
                        docker push ${IMAGE_BASE}:${DEPLOY_ENV}-latest
                    """
                }
            }
        }

        // ── 6. DEPLOY ────────────────────────────────────────
        stage('Deploy to QA') {
            when {
                allOf {
                    branch 'develop'
                    not { changeRequest() }
                }
            }
            steps {
                deployBlueGreen('qa', IMAGE_FULL)
            }
        }

        stage('Deploy to Staging') {
            when {
                allOf {
                    branch 'staging'
                    not { changeRequest() }
                }
            }
            steps {
                deployBlueGreen('staging', IMAGE_FULL)
            }
            post {
                success {
                    // Run load test only on staging
                    sh 'k6 run tests/load/baseline.js --env BASE_URL=https://staging.sync.cloudeagle.internal'
                }
            }
        }

        stage('Prod Approval Gate') {
            when {
                allOf {
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                    branch 'main'
                }
            }
            steps {
                script {
                    def approver = input(
                        id: 'prod-gate',
                        message: "Deploy ${IMAGE_TAG} to PRODUCTION?",
                        submitter: 'Deployment-Approvers',
                        parameters: [
                            string(name: 'CHANGE_TICKET', description: 'JIRA change ticket (required)', defaultValue: ''),
                            booleanParam(name: 'DB_MIGRATION_REVIEWED', description: 'DB migration scripts reviewed?', defaultValue: false)
                        ]
                    )
                    if (!approver.DB_MIGRATION_REVIEWED) {
                        error('DB migration review confirmation required before prod deploy.')
                    }
                    env.CHANGE_TICKET = approver.CHANGE_TICKET
                }
            }
        }

        stage('Deploy to Prod') {
            when {
                allOf {
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                    branch 'main'
                }
            }
            steps {
                deployBlueGreen('prod', IMAGE_FULL)
            }
        }

        // ── 7. SMOKE TESTS ───────────────────────────────────
        stage('Smoke Test') {
            when { not { changeRequest() } }
            steps {
                script {
                    def baseUrl = envBaseUrl(DEPLOY_ENV)
                    retry(3) {
                        sh """
                            curl -sf --max-time 10 ${baseUrl}/actuator/health | \\
                                python3 -c "import sys,json; s=json.load(sys.stdin)['status']; sys.exit(0 if s=='UP' else 1)"
                        """
                    }
                }
            }
        }
    }

    // ── POST ─────────────────────────────────────────────────
    post {
        success {
            script {
                // Record the stable tag for rollback reference
                sh "echo ${IMAGE_TAG} > .last_stable_tag && gsutil cp .last_stable_tag gs://cloudeagle-jenkins-state/${APP_NAME}/${DEPLOY_ENV}/last_stable_tag"
            }
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "✅ *${APP_NAME}* `${IMAGE_TAG}` deployed to *${DEPLOY_ENV}* by ${env.GIT_AUTHOR}"
        }
        failure {
            script {
                echo "Pipeline failed — initiating automated rollback to ${LAST_STABLE_TAG}"
                if (DEPLOY_ENV && LAST_STABLE_TAG && LAST_STABLE_TAG != 'none') {
                    deployBlueGreen(DEPLOY_ENV, "${IMAGE_BASE}:${LAST_STABLE_TAG}")
                }
            }
            slackSend channel: '#incidents',
                      color: 'danger',
                      message: "🚨 *${APP_NAME}* deploy FAILED on *${DEPLOY_ENV}* (commit `${IMAGE_TAG}`). Auto-rollback to `${LAST_STABLE_TAG}` attempted. <!here>"
        }
        always {
            cleanWs()
        }
    }
}

// ── HELPER FUNCTIONS ─────────────────────────────────────────

def resolveEnv() {
    if (env.BRANCH_NAME == 'develop') return 'qa'
    if (env.BRANCH_NAME == 'staging') return 'staging'
    if (env.BRANCH_NAME == 'main')    return 'prod'
    return 'dev'
}

def readLastStableTag() {
    try {
        return sh(
            script: "gsutil cat gs://cloudeagle-jenkins-state/${APP_NAME}/${resolveEnv()}/last_stable_tag 2>/dev/null || echo 'none'",
            returnStdout: true
        ).trim()
    } catch (e) {
        return 'none'
    }
}

def envBaseUrl(String env) {
    def urls = [
        qa      : 'https://qa.sync.cloudeagle.internal',
        staging : 'https://staging.sync.cloudeagle.internal',
        prod    : 'https://sync.cloudeagle.io'
    ]
    return urls[env] ?: 'http://localhost:8080'
}

/**
 * Blue/Green deployment using GCP Managed Instance Groups
 * 1. Create a new (green) MIG with the new image
 * 2. Register green MIG in the backend service at 0% weight
 * 3. Health-check green
 * 4. Shift traffic: 10% canary → 100%
 * 5. Drain and delete old (blue) MIG
 */
def deployBlueGreen(String targetEnv, String imageUri) {
    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
        sh """
            set -euo pipefail
            gcloud auth activate-service-account --key-file=\${GOOGLE_APPLICATION_CREDENTIALS}
            gcloud config set project ${GCP_PROJECT}

            ENV=${targetEnv}
            APP=${APP_NAME}
            IMAGE=${imageUri}
            REGION=asia-south1
            TIMESTAMP=\$(date +%Y%m%d%H%M%S)
            GREEN_MIG="\${APP}-\${ENV}-green-\${TIMESTAMP}"
            BACKEND_SVC="\${APP}-\${ENV}-backend"
            BLUE_MIG=\$(gcloud compute backend-services describe \${BACKEND_SVC} \\
                --region=\${REGION} --format='value(backends[0].group)' | xargs basename 2>/dev/null || echo "")

            echo "==> Creating green MIG: \${GREEN_MIG}"
            gcloud compute instance-groups managed create \${GREEN_MIG} \\
                --region=\${REGION} \\
                --template=\$(gcloud compute instance-templates describe "\${APP}-\${ENV}-template" \\
                    --format='value(selfLink)') \\
                --size=2 \\
                --update-policy-type=PROACTIVE \\
                --metadata=docker-image=\${IMAGE},env=\${ENV}

            echo "==> Waiting for green instances to be stable"
            gcloud compute instance-groups managed wait-until \${GREEN_MIG} \\
                --version-target-reached --region=\${REGION} --timeout=300

            echo "==> Adding green MIG to backend service (0% traffic)"
            gcloud compute backend-services add-backend \${BACKEND_SVC} \\
                --instance-group=\${GREEN_MIG} \\
                --instance-group-region=\${REGION} \\
                --balancing-mode=UTILIZATION \\
                --global

            echo "==> Canary: shifting 10% traffic to green"
            gcloud compute backend-services update-backend \${BACKEND_SVC} \\
                --instance-group=\${GREEN_MIG} \\
                --instance-group-region=\${REGION} \\
                --weight=10 \\
                --global
            sleep 300  # 5 min canary observation

            echo "==> Full cut-over: 100% traffic to green"
            gcloud compute backend-services update-backend \${BACKEND_SVC} \\
                --instance-group=\${GREEN_MIG} \\
                --instance-group-region=\${REGION} \\
                --weight=100 \\
                --global

            if [ -n "\${BLUE_MIG}" ]; then
                echo "==> Draining blue MIG: \${BLUE_MIG}"
                gcloud compute backend-services remove-backend \${BACKEND_SVC} \\
                    --instance-group=\${BLUE_MIG} \\
                    --instance-group-region=\${REGION} \\
                    --global
                sleep 30  # connection drain
                gcloud compute instance-groups managed delete \${BLUE_MIG} \\
                    --region=\${REGION} --quiet
            fi

            echo "==> Blue/Green deploy complete: \${IMAGE} is live on \${ENV}"
        """
    }
}
