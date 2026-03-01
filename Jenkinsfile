pipeline {
  agent any

  options {
    timestamps()
    skipDefaultCheckout(true)
  }

  environment {
    REPO_URL        = "https://github.com/pablotose/todo-list-aws-cp1-4.git"
    CONFIG_REPO_URL = "https://github.com/pablotose/todo-list-aws-config.git"

    REGION     = "us-east-1"
    STACK_NAME = "todo-list-aws"

    VENV       = ".venv"
    REPORT_DIR = "reports"
  }

  stages {

    stage('Get Code') {
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Get Code) =="
          echo "BRANCH_NAME=$BRANCH_NAME"
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "==========================="
        '''

        // ✅ Multibranch: checkout de LA RAMA que está construyendo Jenkins
        checkout(scm)

        // Config repo: staging para develop, production para master
        sh '''#!/bin/bash
          set -euxo pipefail

          if [ "$BRANCH_NAME" = "master" ]; then
            CONF_BRANCH="production"
          elif [ "$BRANCH_NAME" = "develop" ]; then
            CONF_BRANCH="staging"
          else
            # Si te sale alguna rama extra, decide qué quieres aquí:
            CONF_BRANCH="staging"
          fi

          rm -rf config-repo
          git clone -b "$CONF_BRANCH" "${CONFIG_REPO_URL}" config-repo
          cp config-repo/samconfig.toml .
          echo "== samconfig.toml ($CONF_BRANCH) =="
          cat samconfig.toml
        '''

        stash name: 'src', includes: '**/*'
      }
    }

    // =========================
    // CI (develop)
    // =========================
    stage('Static Test (agent: static)') {
      when { branch 'develop' }
      agent { label 'static' }
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Static Test) =="
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "================================"
        '''

        deleteDir()
        unstash 'src'

        sh '''#!/bin/bash
          set -euxo pipefail

          python3 -m venv "${VENV}"
          source "${VENV}/bin/activate"

          pip install --upgrade pip
          pip install flake8 bandit

          mkdir -p "${REPORT_DIR}/flake8"
          mkdir -p "${REPORT_DIR}/bandit"

          flake8 src --exit-zero --tee --output-file "${REPORT_DIR}/flake8/flake8.txt" || true
          bandit -r src -f txt -o "${REPORT_DIR}/bandit/bandit.txt" || true
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', fingerprint: true
        }
      }
    }

    stage('Deploy (Staging)') {
      when { branch 'develop' }
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Deploy Staging) =="
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "================================="
        '''

        deleteDir()
        unstash 'src'

        sh '''#!/bin/bash
          set -euxo pipefail

          rm -rf .aws-sam

          sam validate --region "${REGION}"
          sam build

          sam deploy \
            --region "${REGION}" \
            --stack-name "${STACK_NAME}" \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset

          BASE_URL=$(aws cloudformation describe-stacks \
            --region "${REGION}" \
            --stack-name "${STACK_NAME}" \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BASE_URL=${BASE_URL}" > base_url.env
          echo "API Base URL: ${BASE_URL}"
        '''

        stash name: 'baseurl', includes: 'base_url.env'
      }
    }

    stage('Rest Test (CRUD) (agent: rest)') {
      when { branch 'develop' }
      agent { label 'rest' }
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Rest Test) =="
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "============================"
        '''

        deleteDir()
        unstash 'src'
        unstash 'baseurl'

        sh '''#!/bin/bash
          set -e
          source base_url.env

          echo "Testing API: ${BASE_URL}"
          echo ""

          RESP_POST=$(curl -sf --max-time 15 -X POST "${BASE_URL}/todos" \
            -H "Content-Type: application/json" \
            -d '{ "text": "Test desde Jenkins" }')

          echo "POST response: $RESP_POST"

          TODO_ID=$(python3 - "$RESP_POST" <<'PY'
import json, sys
raw = sys.argv[1].strip()
j = json.loads(raw)
if isinstance(j, dict) and "body" in j and isinstance(j["body"], str):
    j = json.loads(j["body"])
print(j.get("id",""))
PY
)
          echo "TODO_ID=$TODO_ID"
          [ -n "$TODO_ID" ]

          echo "== GET /todos =="
          curl -sS --max-time 15 "${BASE_URL}/todos"
          echo ""

          echo "== GET /todos/${TODO_ID} =="
          curl -sS --max-time 15 "${BASE_URL}/todos/${TODO_ID}"
          echo ""

          echo "== PUT /todos/${TODO_ID} =="
          curl -sS --max-time 15 -X PUT "${BASE_URL}/todos/${TODO_ID}" \
            -H "Content-Type: application/json" \
            -d '{ "text": "Texto actualizado desde Jenkins" }'
          echo ""

          echo "== DELETE /todos/${TODO_ID} =="
          curl -sf --max-time 15 -X DELETE "${BASE_URL}/todos/${TODO_ID}"
          echo ""

          echo "✅ REST TEST PASSED (CRUD completo)"
        '''
      }
    }

    stage('Promote (merge develop -> master)') {
      when { branch 'develop' }
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Promote) =="
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "=========================="
        '''

        // Asegura repo con .git en workspace
        checkout(scm)

        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PAT')]) {
          sh '''#!/bin/bash
            set -euxo pipefail

            git config user.email "jenkins@local"
            git config user.name  "jenkins"

            git fetch origin

            git checkout master || git checkout -b master origin/master
            git pull "https://${GIT_USER}:${GIT_PAT}@github.com/pablotose/todo-list-aws-cp1-4.git" master

            git merge --no-ff origin/develop -m "Promote: merge develop into master"

            git push "https://${GIT_USER}:${GIT_PAT}@github.com/pablotose/todo-list-aws-cp1-4.git" master
          '''
        }
      }
    }

    // =========================
    // CD (master)
    // =========================
    stage('Deploy (Production)') {
      when { branch 'master' }
      steps {
        sh '''#!/bin/bash
          set -e
          echo "== AGENT INFO (Deploy Production) =="
          echo "NODE_NAME=$NODE_NAME"
          echo "EXECUTOR_NUMBER=$EXECUTOR_NUMBER"
          whoami
          hostname
          pwd
          echo "===================================="
        '''

        deleteDir()
        unstash 'src'

        sh '''#!/bin/bash
          set -euxo pipefail

          rm -rf .aws-sam

          sam validate --region "${REGION}"
          sam build

          sam deploy \
            --region "${REGION}" \
            --stack-name "${STACK_NAME}" \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset

          BASE_URL=$(aws cloudformation describe-stacks \
            --region "${REGION}" \
            --stack-name "${STACK_NAME}" \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BASE_URL=${BASE_URL}" > base_url.env
          echo "Production API Base URL: ${BASE_URL}"
        '''

        stash name: 'baseurl', includes: 'base_url.env'
      }
    }

    stage('Rest Test (Read Only) (agent: rest)') {
      when { branch 'master' }
      agent { label 'rest' }
      steps {
        deleteDir()
        unstash 'src'
        unstash 'baseurl'

        sh '''#!/bin/bash
          set -e
          source base_url.env

          echo "Testing production API (read-only): ${BASE_URL}"
          echo ""

          echo "== GET /todos =="
          RESP=$(curl -sS --max-time 15 -w "\\nHTTP:%{http_code}\\n" "${BASE_URL}/todos")
          echo "$RESP"

          HTTP=$(echo "$RESP" | tail -n1 | sed 's/HTTP://')
          if [ "$HTTP" -lt 200 ] || [ "$HTTP" -ge 300 ]; then
            echo "ERROR: GET /todos failed"
            exit 1
          fi

          BODY=$(echo "$RESP" | head -n -1)

          if [ "$BODY" != "[]" ]; then
            ID=$(echo "$BODY" | python3 -c "import sys, json; print(json.load(sys.stdin)[0]['id'])")

            echo ""
            echo "== GET /todos/${ID} =="
            RESP2=$(curl -sS --max-time 15 -w "\\nHTTP:%{http_code}\\n" "${BASE_URL}/todos/${ID}")
            echo "$RESP2"

            HTTP2=$(echo "$RESP2" | tail -n1 | sed 's/HTTP://')
            if [ "$HTTP2" -lt 200 ] || [ "$HTTP2" -ge 300 ]; then
              echo "ERROR: GET /todos/{id} failed"
              exit 1
            fi
          else
            echo ""
            echo "No hay TODOs en producción; se omite GET /todos/{id} (solo lectura)."
          fi

          echo ""
          echo "✅ REST READ-ONLY TEST PASSED"
        '''
      }
    }
  }

  post {
    always {
      echo "Pipeline finished with status: ${currentBuild.currentResult}"
    }
  }
}
