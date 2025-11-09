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
        BACKUP_DIR = "backup-${BUILD_NUMBER}-${GIT_COMMIT?.take(7) ?: 'unknown'}"
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Setup & Build') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    retry(2) {
                        sh '''
                            node --version
                            npm ci
                            npm run build
                        '''
                    }
                }
            }
        }

        stage('Backup to S3') {
            steps {
                script {
                    try {
                        sh """
                            rm -rf backup-tmp
                            mkdir -p backup-tmp
                            
                            echo "Creating backup from FTP..."
                            lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                            mirror /var/www/html/ backup-tmp/
                            exit
                            EOF
                            
                            echo "Uploading backup to S3..."
                            aws s3 cp backup-tmp s3://$S3_BUCKET/backups/${BACKUP_DIR}/ --recursive
                            echo "Backup created at: s3://$S3_BUCKET/backups/${BACKUP_DIR}/"
                        """
                    } catch (Exception e) {
                        slackSend(channel: env.SLACK_CHANNEL, color: 'danger', 
                                message: "Backup failed: ${e.message}")
                        error "Backup failed: ${e.message}"
                    }
                }
            }
        }

        stage('Approval') {
            steps {
                slackSend(channel: env.SLACK_CHANNEL, color: 'warning',
                         message: "Waiting for deployment approval - Build #${BUILD_NUMBER}")
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deploy to production?', ok: 'Deploy'
                }
            }
        }

        stage('Deploy to FTP') {
            steps {
                script {
                    try {
                        sh """
                            lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                            mirror -R --delete ./dist/ /var/www/html/
                            exit
                            EOF
                        """
                    } catch (Exception e) {
                        slackSend(channel: env.SLACK_CHANNEL, color: 'danger',
                                message: "Deployment failed: ${e.message}")
                        error "Deployment failed: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: env.SLACK_CHANNEL, color: 'good',
                     message: """✅ Deploy SUCCESS!
                     Build: #${BUILD_NUMBER}
                     Commit: ${GIT_COMMIT?.take(7) ?: 'unknown'}
                     Branch: ${BRANCH_NAME}""")
        }
        failure {
            script {
                slackSend(channel: env.SLACK_CHANNEL, color: 'danger',
                         message: "⚠️ Deploy FAILED! Starting automatic rollback...")
                
                try {
                    def latestBackup = sh(
                        script: "aws s3 ls s3://$S3_BUCKET/backups/ | sort | tail -1 | awk '{print \$2}'",
                        returnStdout: true
                    ).trim()
                    
                    sh """
                        rm -rf rollback-tmp
                        mkdir -p rollback-tmp
                        aws s3 cp s3://$S3_BUCKET/backups/$latestBackup ./rollback-tmp/ --recursive
                        
                        lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
                        mirror -R --delete ./rollback-tmp/ /var/www/html/
                        exit
                        EOF
                    """
                    
                    slackSend(channel: env.SLACK_CHANNEL, color: 'warning',
                             message: "✅ Rollback successful to backup: ${latestBackup}")
                } catch (Exception e) {
                    slackSend(channel: env.SLACK_CHANNEL, color: 'danger',
                             message: "❌ CRITICAL: Rollback failed: ${e.message}")
                }
            }
        }
        cleanup {
            sh 'rm -rf backup-tmp rollback-tmp'
        }
    }
}
