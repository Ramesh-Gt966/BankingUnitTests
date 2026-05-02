pipeline {
  agent any
  parameters {
    string(name: 'JDK_TOOL', defaultValue: 'jdk-21', description: 'JDK tool name configured on Jenkins for Java 21')
    string(name: 'NODE_IMAGE', defaultValue: 'mcr.microsoft.com/playwright', description: 'Docker image to use for Node/Playwright jobs (optional)')
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Detect project type') {
      steps {
        script {
          nodePresent = fileExists('package.json')
          mavenPresent = fileExists('pom.xml')
          if (!nodePresent && !mavenPresent) {
            error "No `package.json` or `pom.xml` found. Pipeline expects either a Node or Maven project with Playwright tests."
          }
          env.IS_NODE = nodePresent.toString()
          env.IS_MAVEN = mavenPresent.toString()
          echo "IS_NODE=${env.IS_NODE}  IS_MAVEN=${env.IS_MAVEN}"
        }
      }
    }

    stage('Run Playwright (Node)') {
      when {
        expression { return env.IS_NODE == 'true' }
      }
      agent {
        docker {
          image "${params.NODE_IMAGE}"
          args '--shm-size=1g'
        }
      }
      steps {
        echo 'Running Node Playwright tests'
        sh 'npm ci'
        // ensure Playwright browsers and dependencies
        sh 'npx playwright install --with-deps || true'
        sh 'npx playwright test --reporter=dot,html --output=playwright-results || true'
        archiveArtifacts artifacts: 'playwright-results/**', allowEmptyArchive: true
        junit 'playwright-results/**/*.xml'
      }
    }

    stage('Run Playwright (Java/Maven)') {
      when {
        expression { return env.IS_MAVEN == 'true' }
      }
      steps {
        echo 'Running Maven tests (Playwright for Java if present)'
        script {
          // Use JDK tool configured in Jenkins (should point to Java 21)
          def jdkHome = tool name: params.JDK_TOOL, type: 'jdk'
          env.JAVA_HOME = jdkHome
          env.PATH = "${jdkHome}/bin:${env.PATH}"
          echo "Using JAVA_HOME=${env.JAVA_HOME}"
        }
        // Run the Maven test goal. Playwright for Java tests are typically JUnit tests run by surefire.
        sh 'mvn -B test'
        junit 'target/surefire-reports/*.xml'
        // archive common Playwright output locations used by Playwright for Java
        archiveArtifacts artifacts: 'target/playwright-report/**,target/screenshots/**', allowEmptyArchive: true
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished — archiving logs if present'
      archiveArtifacts artifacts: 'target/**/*.log,**/playwright-results/**', allowEmptyArchive: true
    }
    success {
      echo 'Playwright pipeline completed successfully.'
    }
    failure {
      echo 'Playwright pipeline failed. Check the logs and test reports.'
    }
  }
}
