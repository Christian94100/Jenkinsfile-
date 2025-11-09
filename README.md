pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_REGION = 'eu-west-3'
        S3_BUCKET = 'mes-backups-prod'
        SLACK_CHANNEL = '#deploys'
        FTP_HOST = 'ftp.monclient.com'
        FTP_USER = credentials('ftp-user')
        FTP_PASS = credentials('ftp-pass')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm  // Récupère le code Git
            }
        }

        stage('Setup & Build') {
            steps {
                sh '''
                    node --version
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Backup to S3') {
            steps {
                script {
                    def backupDir = "backup-${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    sh """
                        mkdir -p backup-tmp
                        # Backup FTP actuel
                        lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                        mirror /var/www/html/ backup-tmp/
                        exit
                        EOF
                        # Upload S3
                        aws s3 cp backup-tmp s3://$S3_BUCKET/backups/$backupDir/ --recursive
                        echo "Backup: s3://$S3_BUCKET/backups/$backupDir/"
                    """
                }
            }
        }

        stage('Deploy to FTP') {
            steps {
                sh """
                    lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                    mirror -R --delete ./dist/ /var/www/html/
                    exit
                    EOF
                """
            }
        }
    }

    post {
        success {
            slackSend(channel: env.SLACK_CHANNEL, color: 'good', message: "Deploy PROD REUSSI ! Build #${env.BUILD_NUMBER} - Commit ${env.GIT_COMMIT.take(7)}")
        }
        failure {
            script {
                // Rollback auto
                def latestBackup = sh(script: "aws s3 ls s3://$S3_BUCKET/backups/ | tail -1 | awk '{print \$2}'", returnStdout: true).trim()
                sh """
                    aws s3 cp s3://$S3_BUCKET/backups/$latestBackup ./rollback-tmp/ --recursive
                    lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                    mirror -R --delete ./rollback-tmp/ /var/www/html/
                    exit
                    EOF
                """
            }
            slackSend(channel: env.SLACK_CHANNEL, color: 'danger', message: "DEPLOY ÉCHOUÉ ! Build #${env.BUILD_NUMBER} - Rollback auto lancé.")
        }
    }
}
