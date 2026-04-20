pipeline {
    agent any

    environment {
        TRIVY_CACHE_DIR = "/var/lib/jenkins/.trivy"
        TMPDIR = "/var/lib/jenkins/tmp"
    }

    options {
        timestamps()
    }

    stages {
        stage('Prepare Environment') {
            steps {
                sh '''
                echo "Preparing directories..."
                mkdir -p $TRIVY_CACHE_DIR
                mkdir -p $TMPDIR
                mkdir -p reports
                '''
            }
        }

        stage('Initialize Trivy DB') {
            steps {
                sh '''
                echo "Ensuring Vulnerability and Java DBs are present..."
                # Use --cache-dir to ensure it hits your specific path
                # download-db-only and download-java-db-only ensure the environment is ready
                trivy --cache-dir $TRIVY_CACHE_DIR image --download-db-only
                trivy --cache-dir $TRIVY_CACHE_DIR image --download-java-db-only
                '''
            }
        }

        stage('Verify Docker Access') {
            steps {
                sh '''
                echo "Checking Docker access..."
                docker ps
                '''
            }
        }

        stage('List Local Images') {
            steps {
                sh '''
                echo "Available Docker images:"
                docker images
                '''
            }
        }

        stage('Scan Local Images with Trivy') {
            steps {
                sh '''
                set -e

                # Get all image names excluding the headers and <none> tags
                IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>")

                if [ -z "$IMAGES" ]; then
                    echo "No valid Docker images found!"
                    exit 1
                fi

                for image in $IMAGES; do
                    echo "========================================"
                    echo "Scanning image: $image"
                    echo "========================================"

                    SAFE_NAME=$(echo $image | tr '/:' '_')

                    # Table report to console and file
                    trivy --cache-dir $TRIVY_CACHE_DIR image \
                      --timeout 20m \
                      --scanners vuln \
                      --skip-db-update \
                      --skip-java-db-update \
                      --severity HIGH,CRITICAL \
                      --format table \
                      $image | tee reports/${SAFE_NAME}.txt

                    # CSV report using template
                    trivy --cache-dir $TRIVY_CACHE_DIR image \
                      --timeout 20m \
                      --scanners vuln \
                      --skip-db-update \
                      --skip-java-db-update \
                      --format template \
                      --template "@/usr/local/share/trivy/templates/csv.tpl" \
                      -o reports/${SAFE_NAME}.csv \
                      $image

                    echo "Completed scan for $image"
                done
                '''
            }
        }

        stage('Disk Usage After Scan') {
            steps {
                sh '''
                echo "Disk usage after scan:"
                df -h
                '''
            }
        }

        stage('Cleanup (Free Space)') {
            steps {
                sh '''
                echo "Cleaning old Trivy cache..."
                # Clean up general cache but keep the DBs for the next run
                trivy --cache-dir $TRIVY_CACHE_DIR clean --all || true
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'reports/*', fingerprint: true
            }
        }
    }

    post {
        success {
            echo "Trivy scan completed successfully"
        }
        failure {
            echo "Trivy scan failed"
        }
        always {
            echo "Pipeline execution finished"
        }
    }
}
