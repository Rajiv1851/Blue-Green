pipeline {
    agent any

    environment {
        AWS_REGION    = 'ap-south-1'
        ALB_ARN       = 'arn:aws:elasticloadbalancing:ap-south-1:471142785243:loadbalancer/app/blue-green-ALB/297e4faa75d5524f'
        LISTENER_ARN  = 'arn:aws:elasticloadbalancing:ap-south-1:471142785243:listener/app/blue-green-ALB/297e4faa75d5524f/166763c0dbb9d0ac'
        BLUE_TG_ARN   = 'arn:aws:elasticloadbalancing:ap-south-1:471142785243:targetgroup/blue-tg/ffc4e97d24346583'
        GREEN_TG_ARN  = 'arn:aws:elasticloadbalancing:ap-south-1:471142785243:targetgroup/green-tg/dc3c659143d9632a'
        BLUE_IP       = '172.31.45.120'
        GREEN_IP      = '172.31.44.234'
        SSH_KEY       = '/var/lib/jenkins/blue-green-key.pem'
        APP_DIR       = '/usr/share/nginx/html'
    }

    stages {

        stage('Detect Active Environment') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        def activeTG = sh(
                            script: """aws elbv2 describe-listeners \
                                --listener-arns ${env.LISTENER_ARN} \
                                --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                                --output text""",
                            returnStdout: true
                        ).trim()

                        if (activeTG.contains('blue-tg')) {
                            env.ACTIVE      = 'blue'
                            env.INACTIVE    = 'green'
                            env.DEPLOY_IP   = env.GREEN_IP
                            env.NEW_TG      = env.GREEN_TG_ARN
                            env.OLD_TG      = env.BLUE_TG_ARN
                        } else {
                            env.ACTIVE      = 'green'
                            env.INACTIVE    = 'blue'
                            env.DEPLOY_IP   = env.BLUE_IP
                            env.NEW_TG      = env.BLUE_TG_ARN
                            env.OLD_TG      = env.GREEN_TG_ARN
                        }
                        echo "Currently LIVE: ${env.ACTIVE}"
                        echo "Deploying to:   ${env.INACTIVE}"
                    }
                }
            }
        }

        stage('Pull Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Rajiv1851/Blue-Green.git'
                echo "Code pulled from GitHub successfully"
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    sh """
                        scp -o StrictHostKeyChecking=no \
                            -i ${env.SSH_KEY} \
                            -r ./app/* \
                            ec2-user@${env.DEPLOY_IP}:${env.APP_DIR}/
                    """
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            -i ${env.SSH_KEY} \
                            ec2-user@${env.DEPLOY_IP} \
                            'sudo systemctl restart nginx'
                    """
                    echo "Deployed to ${env.INACTIVE}. Waiting 15 seconds..."
                    sleep(15)
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    def healthy = false
                    for (int i = 1; i <= 5; i++) {
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://${env.DEPLOY_IP}/health",
                            returnStdout: true
                        ).trim()
                        echo "Health check attempt ${i}/5 — HTTP status: ${status}"
                        if (status == '200') {
                            healthy = true
                            echo "Health check PASSED!"
                            break
                        }
                        sleep(10)
                    }
                    if (!healthy) {
                        error("Health check FAILED. Starting rollback...")
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws elbv2 modify-listener \
                                --listener-arn ${env.LISTENER_ARN} \
                                --default-actions Type=forward,TargetGroupArn=${env.NEW_TG}
                        """
                    }
                    echo "SUCCESS! Traffic switched to ${env.INACTIVE}"
                }
            }
        }
    }

    post {
        failure {
            script {
                echo "PIPELINE FAILED — Rolling back to ${env.ACTIVE}"
                withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                    sh """
                        aws elbv2 modify-listener \
                            --listener-arn ${env.LISTENER_ARN} \
                            --default-actions Type=forward,TargetGroupArn=${env.OLD_TG}
                    """
                }
                echo "ROLLBACK DONE — ${env.ACTIVE} is still serving traffic"
            }
        }
        success {
            echo "Deployment complete! ${env.INACTIVE} is now live."
        }
    }
}
