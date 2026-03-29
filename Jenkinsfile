pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_IN_AUTOMATION   = 'true'
        SKIP_PIPELINE      = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def changedFiles = sh(
                        script: '''
                            if [ "$(git rev-list --count HEAD)" -eq 1 ]; then
                                git ls-tree --name-only -r HEAD
                            else
                                git diff --name-only HEAD~1 HEAD
                            fi
                        ''',
                        returnStdout: true
                    ).trim().split('\n') as List

                    echo "Changed files: ${changedFiles}"

                    def nonDocChanges = changedFiles.findAll { file ->
                        file?.trim() &&
                        file != 'README.md' &&
                        !file.startsWith('images/')
                    }

                    if (nonDocChanges.isEmpty()) {
                        env.SKIP_PIPELINE = 'true'
                        currentBuild.description = 'Skipped: README/images only'
                        echo 'Only README.md or images/ changed. Skipping pipeline.'
                    }
                }
            }
        }

        stage('Terraform Init') {
            when {
                expression { env.SKIP_PIPELINE != 'true' }
            }
            steps {
                sh 'terraform init -reconfigure'
            }
        }

        stage('Terraform Apply') {
            when {
                expression { env.SKIP_PIPELINE != 'true' }
            }
            steps {
                sh '''
                    terraform plan -out=tfplan
                    terraform apply -auto-approve tfplan
                '''
            }
        }

        stage('Optional Destroy') {
            when {
                expression { env.SKIP_PIPELINE != 'true' }
            }
            steps {
                script {
                    def destroyChoice = input(
                        message: 'Do you want to run terraform destroy?',
                        ok: 'Submit',
                        parameters: [
                            choice(
                                name: 'DESTROY',
                                choices: ['no', 'yes'],
                                description: 'Select yes to destroy resources'
                            )
                        ]
                    )
                    if (destroyChoice == 'yes') {
                        sh 'terraform destroy -auto-approve'
                    } else {
                        echo "Skipping destroy"
                    }
                }
            }
        }
    }
}