pipeline {
  agent any

  options {
    timestamps()
  }

  environment {
    // Ajusta esto si tu repo tiene otro nombre. Debido a que he tenido que crear otro repositorio.
    REPO_URL   = "https://github.com/pablotose/todo-list-aws-cp1-3.git"

    REGION     = "us-east-1"
    STACK_NAME = "todo-list-aws-staging"
    STAGE      = "Staging"

    VENV       = ".venv"
    REPORT_DIR = "reports"
  }

  stages {

    stage('Get Code') {
      steps {
        // En multibranch normalmente bastaría "checkout scm".
        // Aquí lo dejamos explícito para asegurar develop (como pide el enunciado).
        checkout([$class: 'GitSCM',
          branches: [[name: '*/develop']],
          userRemoteConfigs: [[
            url: "${REPO_URL}",
            credentialsId: 'github-token'
          ]]
        ])
      }
    }

    stage('Prepare Env') {
      steps {
        sh '''#!/bin/bash
          set -euxo pipefail

          python3 -m venv "${VENV}"
          source "${VENV}/bin/activate"

          pip install --upgrade pip
          pip install flake8 bandit pytest boto3

          # Si el repo trae requirements, instalarlos
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi

          mkdir -p "${REPORT_DIR}/flake8"
          mkdir -p "${REPORT_DIR}/bandit"
        '''
      }
    }

    stage('Static Test') {
      steps {
        sh '''#!/bin/bash
          set -euxo pipefail
          source "${VENV}/bin/activate"

          # SOLO src/ (como pide la guía)
          # No quality gates: aunque haya findings, no falla (exit-zero / || true)
          flake8 src --exit-zero --tee --output-file "${REPORT_DIR}/flake8/flake8.txt" || true
          bandit -r src -f txt -o "${REPORT_DIR}/bandit/bandit.txt" || true
        '''
      }
      post {
        always {
          // Publica los ficheros como artifacts (válido para entrega)
          archiveArtifacts artifacts: 'reports/**', fingerprint: true
        }
      }
    }

    stage('Deploy (Staging)') {
      steps {
        sh '''#!/bin/bash
          set -euxo pipefail

          sam build
          sam validate --region "${REGION}"

          # Deploy NO guiado (obligatorio automatizado)
          sam deploy \
	    ## Esta conf es porque me esta dando error del bucket de s3, prueba para comprobar que no venga del fichero samconfig ara el --guided
            --no-config-file \
            --stack-name "${STACK_NAME}" \
            --region "${REGION}" \
            --capabilities CAPABILITY_IAM \
            --no-confirm-changeset \
            --resolve-s3 \
            --no-fail-on-empty-changeset \
            --parameter-overrides Stage="${STAGE}"

          # Obtener BaseUrlApi del stack (Outputs)
          BASE_URL=$(aws cloudformation describe-stacks \
            --region "${REGION}" \
            --stack-name "${STACK_NAME}" \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BASE_URL=${BASE_URL}" | tee base_url.env
          echo "API Base URL: ${BASE_URL}"
        '''
      }
    }

    stage('Rest Test') {
      steps {
        sh '''#!/bin/bash
          set -euxo pipefail
          source "${VENV}/bin/activate"
          source base_url.env

          echo "Testing against: ${BASE_URL}"

          # El test puede esperar una env var. Si fuera otra, lo ajustamos.
          export BASE_URL="${BASE_URL}"

          pytest -q test/integration/todoApiTest.py --disable-warnings
        '''
      }
    }

    stage('Promote') {
      when {
        branch 'develop'
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PAT')]) {
          sh '''#!/bin/bash
            set -euxo pipefail

            git config user.email "jenkins@local"
            git config user.name  "jenkins"

            # Asegura tener refs al día
            git fetch origin

            # Cambiar a master
            git checkout master || git checkout -b master origin/master

            # Pull master autenticado
            git pull "https://${GIT_USER}:${GIT_PAT}@github.com/pablotose/todo-list-aws-cp1-3.git" master

            # Merge develop -> master
            git merge --no-ff develop -m "Promote: merge develop into master"

            # Push master
            git push "https://${GIT_USER}:${GIT_PAT}@github.com/pablotose/todo-list-aws-cp1-3.git" master
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished with status: ${currentBuild.currentResult}"
    }
  }
}
