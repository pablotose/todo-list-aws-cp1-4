pipeline {
  agent any

  options {
    timestamps()
  }

  environment {
    REGION     = "us-east-1"
    STACK_NAME = "todo-list-aws-staging"
    STAGE      = "Staging"
    VENV       = ".venv"
  }

  stages {

    stage('Get Code') {
      steps {
        checkout scm
      }
    }

    stage('Prepare Env') {
      steps {
        sh '''
          set -euxo pipefail
          python3 -m venv ${VENV}
          . ${VENV}/bin/activate
          pip install --upgrade pip
          pip install flake8 bandit pytest boto3
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
          mkdir -p reports/flake8
          mkdir -p reports/bandit
        '''
      }
    }

    stage('Static Test') {
      steps {
        sh '''
          set -euxo pipefail
          . ${VENV}/bin/activate

          # Flake8 solo src (no falla el stage aunque haya findings)
          flake8 src --exit-zero --tee --output-file reports/flake8/flake8.txt || true

          # Bandit solo src (no falla el stage aunque haya findings)
          bandit -r src -f txt -o reports/bandit/bandit.txt || true
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'reports/**', fingerprint: true
        }
      }
    }

    stage('Deploy (Staging)') {
      steps {
        sh '''
          set -euxo pipefail

          sam build
          sam validate

          sam deploy \
            --stack-name ${STACK_NAME} \
            --region ${REGION} \
            --capabilities CAPABILITY_IAM \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --parameter-overrides Stage=${STAGE}

          BASE_URL=$(aws cloudformation describe-stacks \
            --region ${REGION} \
            --stack-name ${STACK_NAME} \
            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
            --output text)

          echo "BASE_URL=${BASE_URL}" | tee base_url.env
        '''
      }
    }

    stage('Rest Test') {
      steps {
        sh '''
          set -euxo pipefail
          . ${VENV}/bin/activate
          source base_url.env

          echo "Testing against: ${BASE_URL}"
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
          sh '''
            set -euxo pipefail
            git config user.email "jenkins@local"
            git config user.name  "jenkins"

            git fetch origin
            git checkout master
            git pull https://${GIT_USER}:${GIT_PAT}@github.com/${GIT_USER}/todo-list-aws-cp1-3.git master

            git merge --no-ff develop -m "Promote: merge develop into master"

            git push https://${GIT_USER}:${GIT_PAT}@github.com/${GIT_USER}/todo-list-aws-cp1-3.git master
          '''
        }
      }
    }
  }
}
