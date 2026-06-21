pipeline{

    agent any       // Run on any Jenkins agent
    tools{
        maven 'Maven-3.9'
        jdk 'JDK-17'
        // For building the app
    }

    // Pipeline-wide variables
    environment{
        SONAR_SERVER = 'SonarQube'

        SONAR_PROJECT_KEY = 'webgoat-devsecops'

        IMAGE_NAME = 'webgoat-local'
        IMAGE_TAG = "${env.BUILD_NUMBER}"

        DC_REPORT_DIR = 'dependency-check-report'
    }

    // Stages
    stages{
        
        stage('Checkout'){
            steps{
                checkout scm
                echo "Source code pulled successfully — Branch: ${env.GIT_BRANCH}, Commit: ${env.GIT_COMMIT}"
            }
        }

        stage('Secret Scan'){
            steps{
                script{
                    // Pull and run GitLeaks
                    def exitCode = sh(
                        script:'''
                            docker run --rm \
                            -v "${WORKSPACE}:/repo" \
                            zricethezav/gitleaks:latest \
                            detect \
                            --source /repo \
                            --report-format json \
                            --report-path /repo/gitleaks-report.json \
                            --no-git \
                            -v
                        ''',
                        returnStatus: true //prevents failing, handled below
                    )

                    if (exitCode == 1){
                        // Archive the report can be downloaded from Jenkins
                        archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                        // Fail the build so the code inst shipped
                        error("Check gitleaks-report.json")
                    }
                    else {
                        echo "No Secrets"
                    }
                }
            }
        }
        
        stage('Compile') {
            steps {
                // Compile only — no tests yet, just produce target/classes
                // SonarQube needs the .class files to analyze bytecode
                sh 'mvn compile -DskipTests'
            }
        }


        stage('SAST: SonarQube analysis'){
            steps{
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName='WebGoat DevSecOps Demo' \
                          -Dsonar.sources=src/main/java \
                          -Dsonar.tests=src/test/java \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }

            }
        }

        stage('SAST: Quality Gate') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('SCA: OWASP Dependency-Check') {
            steps {
                dependencyCheck(
                    additionalArguments: """
                        --scan ./
                        --format HTML
                        --format XML
                        --out ${DC_REPORT_DIR}
                        --enableRetired
                    """,
                    odcInstallation: 'OWASP-DC'
                )
            }
            post {
                always {
                    // Parse the XML
                    dependencyCheckPublisher(
                        pattern: "${DC_REPORT_DIR}/dependency-check-report.xml",
                        failedTotalCritical: 1,    // Fail build if ANY critical CVE found
                        failedTotalHigh: 5,        // Fail if more than 5 high CVEs
                        unstableTotalMedium: 10    // Mark unstable if more than 10 mediums
                    )
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: DC_REPORT_DIR,
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency Check Report'
                    ])
                }
            }
        }
        
        stage('Docker build') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                    echo "Docker image built: ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        // Needs Container scanning (tool like Trivy)

    } 
    // end stages

    // Run regardless of outcome
    post {

        success {
            echo """
               Pipeline PASSED: All security gates cleared
               Image: ${IMAGE_NAME}:${IMAGE_TAG} is ready
            """
        }

        failure {
            echo """
               Pipeline FAILED: Security issue detected
               Check stage logs and archived reports above
            """
        }

        always {
            // Clean up the built Docker image to save disk space
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"

            // Clean Maven build artifacts from workspace
            sh "mvn clean || true"
        }
    }

}
// end pipeline
