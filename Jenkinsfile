def failureMessages = []
def DO_TOKEN_CREDENTIAL_ID = 'digitalocean-ansible-roles-oauth-token'
def DO_SSH_KEYS_CREDENTIAL_ID = 'digitalocean-reserved-ip-ssh-key-ids'
def SSH_PRIVATE_KEY_CREDENTIAL_ID = 'digitalocean-reserved-ip-test-ssh-private-key'
def SLACK_TOKEN_CREDENTIAL_ID = 'inviqa-slack-integration-token'
def TEST_INVENTORY_CHOICES = [
    'tests/inventory',
    'tests/inventory-debian',
    'tests/inventory-centos',
    'tests/inventory-ubuntu'
]

def runWithSshAgent(String command) {
    sshagent(credentials: [SSH_PRIVATE_KEY_CREDENTIAL_ID]) {
        sh command
    }
}

pipeline {
    agent {
        docker {
            label 'linux-amd64'
            alwaysPull true
            image 'quay.io/inviqa_images/ansible:2.15-python3.10-trixie'
            args '--entrypoint=""'
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        ANSIBLE_COLLECTIONS_PATH = ".ansible/collections:/home/ansible/.ansible/collections:/usr/share/ansible/collections"
        ANSIBLE_FORCE_COLOR = 'true'
        ANSIBLE_ROLES_PATH = "tests/roles:.ansible/roles:/home/ansible/.ansible/roles"
        SLACK_NOTIFICATION_CHANNEL = 'ops-integrations'
    }

    parameters {
        booleanParam(
            name: 'RUN_LIVE_TESTS',
            defaultValue: true,
            description: 'Run the DigitalOcean live integration test matrix.'
        )
        choice(
            name: 'TEST_INVENTORY',
            choices: TEST_INVENTORY_CHOICES,
            description: 'Inventory used for the live test and cleanup playbooks.'
        )
    }

    stages {
        stage('Install Ansible dependencies') {
            steps {
                sh 'ansible-galaxy collection install -r tests/requirements.yml -p .ansible/collections'
            }
            post {
                failure {
                    script { failureMessages << 'Ansible collection installation failed' }
                }
            }
        }

        stage('Syntax checks') {
            steps {
                sh 'ansible-playbook --syntax-check -i tests/inventory tests/playbook.yml'
                sh 'ansible-playbook --syntax-check -i tests/inventory tests/playbook_cleanup.yml'
            }
            post {
                failure {
                    script { failureMessages << 'Ansible playbook syntax checks failed' }
                }
            }
        }

        stage('Live DigitalOcean tests') {
            when {
                expression { return params.RUN_LIVE_TESTS }
            }
            steps {
                script {
                    if (!TEST_INVENTORY_CHOICES.contains(params.TEST_INVENTORY)) {
                        error("Unsupported TEST_INVENTORY '${params.TEST_INVENTORY}'")
                    }

                    withCredentials([
                        string(credentialsId: DO_TOKEN_CREDENTIAL_ID, variable: 'DIGITAL_OCEAN_API_TOKEN'),
                        string(credentialsId: DO_SSH_KEYS_CREDENTIAL_ID, variable: 'DO_SSH_KEYS')
                    ]) {
                        sh '''
                            set -eu
                            umask 077
                            {
                                printf '%s\n' '---'
                                printf '%s\n' 'do_ssh_keys:'
                                printf '%s\n' "$DO_SSH_KEYS" |
                                    tr ',' '\\n' |
                                    sed 's/^[[:space:]]*//; s/[[:space:]]*$//' |
                                    awk 'NF { printf "  - \\"%s\\"\\n", $0 }'
                            } > tests/test_variables.yml
                        '''

                        try {
                            runWithSshAgent("ansible-playbook -i '${params.TEST_INVENTORY}' tests/playbook.yml")
                        } finally {
                            runWithSshAgent("ansible-playbook -i '${params.TEST_INVENTORY}' tests/playbook_cleanup.yml")
                        }
                    }
                }
            }
            post {
                failure {
                    script { failureMessages << 'Live DigitalOcean integration tests failed' }
                }
            }
        }
    }

    post {
        failure {
            script {
                def message = "ansible-digitalocean-reserved-ip: ${env.JOB_BASE_NAME} #${env.BUILD_NUMBER} - Failure after ${currentBuild.durationString.minus(' and counting')} (<${env.RUN_DISPLAY_URL}|View Build>)"
                def fallbackMessages = [ message ]
                def fields = []

                def failureMessage = failureMessages.join("\n")
                if (failureMessage) {
                    fields << [
                        title: 'Reason(s)',
                        value: failureMessage,
                        short: false
                    ]
                    fallbackMessages << failureMessage
                }
                def attachments = [
                    [
                        text: message,
                        fallback: fallbackMessages.join("\n"),
                        color: 'danger',
                        fields: fields
                    ]
                ]

                slackSend(channel: env.SLACK_NOTIFICATION_CHANNEL, color: 'danger', attachments: attachments, tokenCredentialId: SLACK_TOKEN_CREDENTIAL_ID)
            }
        }
        always {
            sh 'rm -f tests/test_variables.yml'
            cleanWs()
        }
    }
}
