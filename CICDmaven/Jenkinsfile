pipeline {
    agent any

    tools {
        maven 'maven-3'
        jdk   'jdk-17'
    }

    environment {
        SONARQUBE = "sonarqube"
        NEXUS_USER = "admin"
        NEXUS_PASS = "Nexus1@34"
        NEXUS_URL  = "http://localhost:8081/repository/maven-releases/"
        APP_VERSION = "1.0.0"
        IMAGE_NAME  = "cicd-app"
    }

    stages {

        // ------------------------------------------------------------------
        stage('1. Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/prasanthdeva/CICD.git'
            }
        }

        // ------------------------------------------------------------------
        stage('2. Build & Unit Tests') {
            steps {
                sh "mvn clean package"
            }
        }

        // ------------------------------------------------------------------
        stage('3. SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh "mvn sonar:sonar"
                }
            }
        }

        // ------------------------------------------------------------------
        stage('4. OWASP Dependency Scan') {
            steps {
                sh """
                mvn org.owasp:dependency-check-maven:check \
                    -Dformat=HTML \
                    -DoutputDirectory=dependency-check-report
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*'
                }
            }
        }

        // ------------------------------------------------------------------
        stage('5. SBOM (CycloneDX)') {
            steps {
                sh "mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom"
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/bom.xml'
                }
            }
        }

        // ------------------------------------------------------------------
        stage('6. Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${APP_VERSION} ."
            }
        }

        // ------------------------------------------------------------------
        stage('7. Grype CVE Scan') {
            steps {
                sh """
                grype ${IMAGE_NAME}:${APP_VERSION} -o json > grype-report.json
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'grype-report.json'
                }
            }
        }

        // ------------------------------------------------------------------
        stage('8. Security Gate') {
            steps {
                script {
                    def vulns = sh(
                        script: "cat grype-report.json | jq '.matches | length'",
                        returnStdout: true
                    ).trim()

                    echo "Total vulnerabilities found = ${vulns}"

                    if (vulns.toInteger() > 0) {
                        error("❌ Security Gate Failed — Fix vulnerabilities")
                    } else {
                        echo "✅ Security Gate Passed"
                    }
                }
            }
        }

        // ------------------------------------------------------------------
        stage('9. Upload to Nexus (Hardcoded creds)') {
            steps {
                sh """
                curl -v -u ${NEXUS_USER}:${NEXUS_PASS} \
                    --upload-file target/${IMAGE_NAME}-${APP_VERSION}.jar \
                    ${NEXUS_URL}com/cicd/${IMAGE_NAME}/${APP_VERSION}/${IMAGE_NAME}-${APP_VERSION}.jar
                """
            }
        }

        // ------------------------------------------------------------------
        stage('10. Deploy / Run') {
            steps {
                sh "java -jar target/${IMAGE_NAME}-${APP_VERSION}.jar"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed — Check logs."
        }
    }
}
