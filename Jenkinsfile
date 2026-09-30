pipeline {
    agent any

    environment {
        TZ                   = 'UTC-9'
        DOCKER_REGISTRY      = 'bitholla/dashboard'
        INFRA_REPO           = 'https://github.com/hollaex/infrastructure.git'
        INFRA_SERVER_VALUES  = "${env.INFRA_SERVER_VALUES ?: 'kubernetes/testnet/dashboard/helm-values/values-testnet.yaml'}"
        INFRA_CLIENT_VALUES  = "${env.INFRA_CLIENT_VALUES ?: 'kubernetes/testnet/dashboard/helm-values/values-client-testnet.yaml'}"
        DOCKER_ACCESS_TOKEN  = credentials('DOCKER_ACCESS_TOKEN')
        DOCKER_USERNAME      = credentials('DOCKER_USERNAME')
        GITHUB_ACCESS_TOKEN  = credentials('GITHUB_ACCESS_TOKEN_BITHOLLA_HOLLACLOUD')
    }

    stages {

        stage('Notify Start') {
            steps {
                script {
                    slackNotifyStart("Build started: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
                }
            }
        }

        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "echo ${env.GIT_COMMIT} | cut -c 1-7", returnStdout: true).trim()
                    env.GIT_BRANCH_CLEAN = sh(script: "echo ${env.GIT_BRANCH} | cut -f2 -d '/'", returnStdout: true).trim()

                    env.SERVER_VERSION = sh(script: "jq -r '.version' server/package.json", returnStdout: true).trim()
                    env.CLIENT_VERSION = sh(script: "jq -r '.version' client/package.json", returnStdout: true).trim()

                    env.SERVER_TAG = "server-${env.SERVER_VERSION}-${env.GIT_BRANCH_CLEAN}-${env.GIT_COMMIT_SHORT}"
                    env.CLIENT_TAG    = "client-${env.CLIENT_VERSION}-${env.GIT_BRANCH_CLEAN}-${env.GIT_COMMIT_SHORT}"

                    // Detect which directories changed compared to the previous commit
                    def changes = sh(script: "git diff --name-only HEAD~1 || echo ''", returnStdout: true).trim()
                    def jenkinsfileChanged = changes.contains('Jenkinsfile')

                    env.CLIENT_CHANGED = (jenkinsfileChanged || changes.contains('client/')) ? 'true' : 'false'
                    env.SERVER_CHANGED = (jenkinsfileChanged || changes.contains('server/')) ? 'true' : 'false'
                }

                echo "Server changed: ${env.SERVER_CHANGED} | Client changed: ${env.CLIENT_CHANGED}"
                echo "Server tag: ${env.SERVER_TAG} | Client tag: ${env.CLIENT_TAG}"
            }
        }

        stage('Docker Login') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -eo pipefail
                    LOG=/dev/null

                    # Retry a step that is safe to repeat. Docker Hub logins, pulls and
                    # pushes fail on transient network/5xx blips often enough that a
                    # single extra attempt turns most red builds green, and everything
                    # wrapped here is idempotent: it rebuilds or re-uploads the same tag.
                    retry_step() {
                        local label="$1" attempts="$2"; shift 2
                        local i=1
                        while true; do
                            if "$@"; then
                                return 0
                            fi
                            if [ "${i}" -ge "${attempts}" ]; then
                                echo "${label}: failed after ${i} attempt(s)." | tee -a "${LOG}"
                                return 1
                            fi
                            echo "${label}: attempt ${i}/${attempts} failed; retrying in $((i * 10))s..." | tee -a "${LOG}"
                            sleep "$((i * 10))"
                            i=$((i + 1))
                        done
                    }

                    # --password-stdin keeps the token out of the agent's process list,
                    # which `-p` exposed to anything able to read /proc.
                    registry_login() {
                        printf '%s' "${DOCKER_ACCESS_TOKEN}" \
                            | docker login -u "${DOCKER_USERNAME}" --password-stdin
                    }
                    retry_step "docker login" 3 registry_login
                '''
            }
        }

        stage('Build & Push') {
            parallel {

                stage('Server') {
                    when { expression { env.SERVER_CHANGED == 'true' } }
                    steps {
                        // tee the stage output to its own file so a failure posts
                        // only this stage's log, not the interleaved output of the
                        // parallel Client branch. pipefail keeps docker's exit code.
                        sh '''#!/usr/bin/env bash
                            set -eo pipefail
                            LOG=server-build.log
                            : > "${LOG}"

                            retry_step() {
                                local label="$1" attempts="$2"; shift 2
                                local i=1
                                while true; do
                                    if "$@"; then
                                        return 0
                                    fi
                                    if [ "${i}" -ge "${attempts}" ]; then
                                        echo "${label}: failed after ${i} attempt(s)." | tee -a "${LOG}"
                                        return 1
                                    fi
                                    echo "${label}: attempt ${i}/${attempts} failed; retrying in $((i * 10))s..." | tee -a "${LOG}"
                                    sleep "$((i * 10))"
                                    i=$((i + 1))
                                done
                            }

                            build_image() {
                                docker build --platform=linux/amd64 \
                                    -t "${DOCKER_REGISTRY}:${SERVER_TAG}" \
                                    -f server/Dockerfile server/ 2>&1 | tee -a "${LOG}"
                            }
                            push_image() {
                                docker push "${DOCKER_REGISTRY}:${SERVER_TAG}" 2>&1 | tee -a "${LOG}"
                            }
                            retry_step "server image build" 2 build_image
                            retry_step "server image push" 3 push_image
                        '''
                    }
                    post {
                        failure {
                            script { notifyStageFailure('Server', 'server-build.log') }
                        }
                    }
                }

                stage('Client') {
                    when { expression { env.CLIENT_CHANGED == 'true' } }
                    steps {
                        sh '''#!/usr/bin/env bash
                            set -eo pipefail
                            LOG=client-build.log
                            : > "${LOG}"

                            retry_step() {
                                local label="$1" attempts="$2"; shift 2
                                local i=1
                                while true; do
                                    if "$@"; then
                                        return 0
                                    fi
                                    if [ "${i}" -ge "${attempts}" ]; then
                                        echo "${label}: failed after ${i} attempt(s)." | tee -a "${LOG}"
                                        return 1
                                    fi
                                    echo "${label}: attempt ${i}/${attempts} failed; retrying in $((i * 10))s..." | tee -a "${LOG}"
                                    sleep "$((i * 10))"
                                    i=$((i + 1))
                                done
                            }

                            build_image() {
                                docker build --platform=linux/amd64 \
                                    -t "${DOCKER_REGISTRY}:${CLIENT_TAG}" \
                                    -f client/Dockerfile client/ 2>&1 | tee -a "${LOG}"
                            }
                            push_image() {
                                docker push "${DOCKER_REGISTRY}:${CLIENT_TAG}" 2>&1 | tee -a "${LOG}"
                            }
                            retry_step "client image build" 2 build_image
                            retry_step "client image push" 3 push_image
                        '''
                    }
                    post {
                        failure {
                            script { notifyStageFailure('Client', 'client-build.log') }
                        }
                    }
                }
            }
        }

        stage('Update Infrastructure') {
            when {
                expression { env.SERVER_CHANGED == 'true' || env.CLIENT_CHANGED == 'true' }
            }
            steps {
                script {
                    sh """
                        git config --global user.name "bitholla-Hollacloud"
                        git config --global user.password "\${GITHUB_ACCESS_TOKEN}"
                        git config --global user.email "tech@bitholla.com"
                        git config --global url."https://bitholla-Hollacloud:\${GITHUB_ACCESS_TOKEN}@github.com/".insteadOf "https://github.com/"
                    """

                    dir('infrastructure') {
                        // Without this a second build in the same workspace hits a
                        // non-empty infrastructure/ and `git clone .` refuses to run.
                        deleteDir()
                        sh '''#!/usr/bin/env bash
                            set -eo pipefail
                            LOG=/dev/null

                            retry_step() {
                                local label="$1" attempts="$2"; shift 2
                                local i=1
                                while true; do
                                    if "$@"; then
                                        return 0
                                    fi
                                    if [ "${i}" -ge "${attempts}" ]; then
                                        echo "${label}: failed after ${i} attempt(s)." | tee -a "${LOG}"
                                        return 1
                                    fi
                                    echo "${label}: attempt ${i}/${attempts} failed; retrying in $((i * 5))s..." | tee -a "${LOG}"
                                    sleep "$((i * 5))"
                                    i=$((i + 1))
                                done
                            }

                            clone_infra() {
                                # A half-written clone leaves files behind and `git clone .`
                                # then refuses to run, so start each attempt from an empty dir.
                                find . -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
                                git clone "${INFRA_REPO}" .
                            }
                            retry_step "clone infrastructure" 3 clone_infra
                        '''

                        if (env.SERVER_CHANGED == 'true') {
                            sh "yq e -i '.dockerTag = \"${env.SERVER_TAG}\"' ${INFRA_SERVER_VALUES}"
                            sh "git add ${INFRA_SERVER_VALUES}"
                        }
                        if (env.CLIENT_CHANGED == 'true') {
                            sh "yq e -i '.dockerTag = \"${env.CLIENT_TAG}\"' ${INFRA_CLIENT_VALUES}"
                            sh "git add ${INFRA_CLIENT_VALUES}"
                        }

                        // Re-running a build on an unchanged commit stages nothing, and a
                        // bare `git commit` then exits 1 and fails the whole pipeline over
                        // a no-op. The other repos' pipelines already carry this guard.
                        sh 'git diff --cached --quiet || git commit -m "HollaEx Dashboard Build - server:${SERVER_TAG} client:${CLIENT_TAG}"'
                        sh '''#!/usr/bin/env bash
                            set -eo pipefail
                            LOG=/dev/null

                            retry_step() {
                                local label="$1" attempts="$2"; shift 2
                                local i=1
                                while true; do
                                    if "$@"; then
                                        return 0
                                    fi
                                    if [ "${i}" -ge "${attempts}" ]; then
                                        echo "${label}: failed after ${i} attempt(s)." | tee -a "${LOG}"
                                        return 1
                                    fi
                                    echo "${label}: attempt ${i}/${attempts} failed; retrying in $((i * 5))s..." | tee -a "${LOG}"
                                    sleep "$((i * 5))"
                                    i=$((i + 1))
                                done
                            }

                            push_infra() {
                                # An earlier attempt that hit a rebase conflict leaves the
                                # repo mid-rebase on a detached HEAD; clear that first so
                                # this attempt cannot push a half-rebased tree.
                                git rebase --abort >/dev/null 2>&1 || true
                                # Another repo's pipeline may have pushed to infrastructure
                                # between our clone and this push, which rejects us as
                                # non-fast-forward. Rebasing onto whatever landed and pushing
                                # again is exactly the manual recovery, so do it automatically.
                                #
                                # && rather than a plain sequence: retry_step calls this from
                                # an `if` condition, where bash suppresses `set -e` for the
                                # whole call, so a failed pull would otherwise fall through
                                # and push whatever HEAD the aborted rebase left behind.
                                git pull --rebase && git push
                            }
                            retry_step "push infrastructure" 3 push_infra
                        '''
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                // The Server/Client stages post their own captured log (only the
                // failing stage's output, no parallel interleaving). Skip here if
                // one already did so we don't double-notify. This branch is the
                // fallback for failures in the sequential stages (Prepare, Docker
                // Login, Update Infrastructure), where a whole-log tail is clean.
                if (env.FAILURE_NOTIFIED == 'true') {
                    return
                }

                def text = "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

                // Grab the tail of the console log to attach as a threaded reply.
                // Reading the run log may need a one-time in-process script
                // approval when the pipeline runs in the Groovy sandbox; until
                // then this stays empty and we just post the plain failure line.
                def logTail = ''
                def errorLine = ''
                try {
                    def joined = currentBuild.rawBuild.getLog(150).join('\n')
                    logTail = formatLogTail(joined, 120, 12000)
                    errorLine = lastMeaningfulLine(joined)
                } catch (e) {
                    echo "Warning: could not read build log for Slack thread: ${e.message}"
                }
                // Surface the last meaningful log line in the main message so the
                // actual error is visible in the channel without expanding the
                // thread (Slack collapses long messages to a head preview).
                if (errorLine) {
                    text += "\n> ${errorLine}"
                }
                if (logTail) {
                    text += "\nCheck the thread to see the full failure logs."
                }

                // Broadcast the verdict so it is visible in the channel itself,
                // and keep the log volume inside the thread.
                slackNotifyThread('danger', text, true)
                if (logTail) {
                    slackNotifyThread('danger', "Failure log — last 120 log lines:\n```\n${logTail}\n```", false)
                }
            }
        }
        success {
            script {
                slackNotifyThread('good',
                    "Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} - server:${env.SERVER_TAG} client:${env.CLIENT_TAG} (<${env.BUILD_URL}|Open>)",
                    true)
            }
        }
    }
}

// ─── Slack ────────────────────────────────────────────────────────────────
// One build == one Slack thread. The first message is a normal channel post
// (unchanged from before); everything after it replies underneath, so a build
// no longer scatters separate lines across #deployment.
//
// Threading needs the message id of the parent, and the plugin only hands that
// back when it talks to the Slack Web API as a bot user. `botUser: true` +
// `failOnError: true` makes that path either work or raise — never silently
// drop the message — so the catch below can fall back to the plain channel
// post this pipeline used before. Threading is therefore an improvement, never
// a dependency: if the plugin is not bot-user configured, notifications keep
// working exactly as they do today.
def slackPost(String threadId, String color, String message, boolean broadcast) {
    try {
        return slackSend(
            channel: threadId ?: '#deployment',
            color: color,
            message: message,
            botUser: true,
            replyBroadcast: (threadId && broadcast) ? true : false,
            failOnError: true,
            tokenCredentialId: 'slack-oauth-token'
        )
    } catch (e) {
        echo "Warning: threaded Slack send failed (${e.message}); falling back to a plain channel message."
    }
    try {
        return slackSend(
            channel: '#deployment',
            color: color,
            message: message,
            tokenCredentialId: 'slack-oauth-token'
        )
    } catch (e) {
        echo "Warning: Slack notification failed: ${e.message}"
        return null
    }
}

// The build's opening message. Its id becomes the thread every later message
// replies to; an empty id just means later messages stay in the channel.
def slackNotifyStart(String message) {
    def resp = slackPost('', '#439FE0', message, false)
    env.SLACK_THREAD_ID = resp?.threadId ?: ''
}

// Every message after the first. `broadcast` additionally surfaces the reply in
// the channel — used for the final SUCCESS/FAILED verdict so a result is never
// hidden behind a collapsed thread.
def slackNotifyThread(String color, String message, boolean broadcast) {
    slackPost(env.SLACK_THREAD_ID ?: '', color, message, broadcast)
}

// Post a single stage's captured log to Slack. `logFile` is the file the stage
// tee'd its shell output into, so only the failing stage's output is shown —
// never the interleaved output of a parallel branch. Uses fileExists/readFile
// (both sandbox-safe), so this path needs no script approval.
def notifyStageFailure(String stageName, String logFile) {
    env.FAILURE_NOTIFIED = 'true'

    def logTail = ''
    def errorLine = ''
    if (fileExists(logFile)) {
        def content = readFile(logFile)
        logTail = formatLogTail(content, 120, 12000)
        errorLine = lastMeaningfulLine(content)
    }

    def text = "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} — ${stageName} stage (<${env.BUILD_URL}|Open>)"
    // Surface the last meaningful log line in the main message so the actual
    // error (e.g. `Error: building at STEP "RUN npm run build"...`) is visible
    // in the channel without expanding the thread.
    if (errorLine) {
        text += "\n> ${errorLine}"
    }
    if (logTail) {
        text += "\nCheck the thread to see the full failure logs."
    }

    slackNotifyThread('danger', text, true)
    if (logTail) {
        slackNotifyThread('danger', "${stageName} stage failure log (last 120 lines):\n```\n${logTail}\n```", false)
    }
}

// Reduce a captured build/stage log to a compact, Slack-friendly tail:
//  - Strip ANSI colour codes. buildah, npm and Next.js all emit them; in a
//    Slack code block they render as `[0m`/`[31m` garbage and inflate the
//    byte count for no benefit.
//  - Keep only the last `maxLines` lines. Build failures print the real error
//    at the very END of the log (e.g. `Error: building at STEP "RUN npm run
//    build"...`), while the head is just image pulls and npm-install noise —
//    so tailing by line is what actually surfaces the cause. This runs
//    unconditionally (not only when the log is huge): posting the whole log
//    let Slack collapse it to a head-only preview, hiding the trailing error.
//  - Hard-cap the result keeping the END, so the reply stays well under
//    Slack's ~40k-char message limit even for a runaway log.
// Pure string work over fileExists/readFile output, so no script approval.
def formatLogTail(String content, int maxLines, int maxChars) {
    if (!content) {
        return ''
    }
    content = content.replaceAll(/\x1b\[[0-9;?]*[A-Za-z]/, '').trim()
    def lines = content.split('\n')
    if (lines.length > maxLines) {
        content = lines[(lines.length - maxLines)..-1].join('\n')
    }
    if (content.length() > maxChars) {
        content = '...(truncated)...\n' + content.substring(content.length() - maxChars)
    }
    return content
}

// The last non-empty line of a log (ANSI stripped) — the single most useful
// summary line, which for buildah/npm failures is the terminal error. Surfaced
// at the top of the Slack message so it shows even when Slack collapses the
// threaded code block to a preview.
def lastMeaningfulLine(String content) {
    if (!content) {
        return ''
    }
    def stripped = content.replaceAll(/\x1b\[[0-9;?]*[A-Za-z]/, '')
    def result = ''
    for (line in stripped.split('\n')) {
        if (line?.trim()) {
            result = line.trim()
        }
    }
    return result
}
