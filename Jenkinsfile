pipeline {
    agent any

    options {
        // Keep only last 15 builds or 15 days
        buildDiscarder(logRotator(
            daysToKeepStr: '15',
            numToKeepStr: '15'
        ))

        // Prevent parallel executions of the same job
        disableConcurrentBuilds()
    }

    parameters {
        // Angular environment (matches angular.json configurations)
        choice(
            name: 'ENVIRONMENT',
            choices: [
                'dev',
                'qa',
                'stage',
                'prod'
            ],
            description: 'Angular build environment'
        )

        // Git branch to build
        string(
            name: 'BRANCH_NAME',
            defaultValue: 'main',
            description: 'Git branch to build and deploy'
        )
    }

    environment {
        // CHANGE_ME: Your S3 bucket (must already exist)
        S3_BUCKET = "s3://your-frontend-bucket-name"

        // CHANGE_ME: CloudFront Distribution ID
        CLOUDFRONT_DIST_ID = "YOUR_CLOUDFRONT_DISTRIBUTION_ID"

        // CHANGE_ME (Optional): Node memory limit for Angular builds
        NODE_OPTIONS = "--max_old_space_size=8000"
    }

    stages {

        stage("Checkout Source Code") {
            steps {
                echo "Building branch: ${params.BRANCH_NAME}"

                // CHANGE_ME:
                // - credentialsId
                // - Git repository URL
                git branch: params.BRANCH_NAME,
                    credentialsId: 'github-credentials-id',
                    url: 'https://github.com/your-org/your-angular-repo.git'
            }
        }

        stage("Install Dependencies (Smart Install)") {
            steps {
                sh '''
                    set -e
                    echo "Checking dependency fingerprint..."

                    md5sum package.json package-lock.json > .deps.current

                    NEED_INSTALL=true

                    if [ -d node_modules ] && [ -f .deps.previous ]; then
                        if diff .deps.current .deps.previous >/dev/null; then
                            NEED_INSTALL=false
                        fi
                    fi

                    if [ "$NEED_INSTALL" = true ]; then
                        echo "Dependencies changed – running npm install"
                        npm install
                        cp .deps.current .deps.previous
                    else
                        echo "Dependencies unchanged – skipping npm install"
                    fi
                '''
            }
        }

        stage("Build Angular Application") {
            steps {
                sh '''
                    echo "Building Angular for environment: ${ENVIRONMENT}"

                    node --openssl-legacy-provider \
                      ./node_modules/@angular/cli/bin/ng build \
                      --configuration=${ENVIRONMENT}
                '''
            }
        }

        stage("Verify Build Output") {
            steps {
                sh '''
                    echo "Build output:"
                    ls -lh dist || exit 1
                '''
            }
        }

        stage("Deploy to S3 (Full Replace)") {
            steps {
                sh '''
                    echo "Deploying to S3 bucket: ${S3_BUCKET}"

                    aws s3 sync dist/ ${S3_BUCKET} --delete
                '''
            }
        }

        stage("Invalidate CloudFront Cache") {
            steps {
                sh '''
                    echo "Invalidating CloudFront distribution: ${CLOUDFRONT_DIST_ID}"

                    aws cloudfront create-invalidation \
                      --distribution-id ${CLOUDFRONT_DIST_ID} \
                      --paths "/*"
                '''
            }
        }
    }

    post {
        success {
            echo "Frontend (${ENVIRONMENT}) deployed successfully"
        }
        failure {
            echo "Build or deployment failed"
        }
    }
}
