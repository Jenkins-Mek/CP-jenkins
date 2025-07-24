//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Name of the Kafka topic to delete'),
        booleanParam(name: 'CONFIRM_DELETE', defaultValue: false, description: 'Confirm deletion - WARNING: This action is irreversible!')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME parameter is required to delete a topic.")
                    }

                    if (!params.CONFIRM_DELETE) {
                        error("❌ CONFIRM_DELETE must be checked to proceed with topic deletion. This action is irreversible!")
                    }

                    echo "⚠️ WARNING: About to delete topic '${params.TOPIC_NAME}' - This action cannot be undone!"
                }
            }
        }

        stage('Create Kafka Client Config') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                       createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Verify Topic Exists') {
            steps {
                script {
                    def topicName = params.TOPIC_NAME.trim()
                    echo "🔍 Checking if topic '${topicName}' exists..."
                    def exists = verifyTopicExists(topicName)
                    if (!exists) {
                        error("❌ Topic '${topicName}' does not exist. Cannot delete non-existent topic.")
                    }
                    echo "✅ Topic '${topicName}' found and ready for deletion."
                }
            }
        }

        stage('Delete Topic') {
            steps {
                script {
                    def topicName = params.TOPIC_NAME.trim()
                    echo "🗑️ Deleting Kafka topic: ${topicName}"
                    def result = deleteKafkaTopic(topicName)
                    echo result
                }
            }
        }

        stage('Verify Deletion') {
            steps {
                script {
                    def topicName = params.TOPIC_NAME.trim()
                    echo "🔍 Verifying topic '${topicName}' has been deleted..."
                    sleep(time: 5, unit: 'SECONDS') // Wait a bit for deletion to propagate
                    def stillExists = verifyTopicExists(topicName)
                    if (stillExists) {
                        echo "⚠️ Topic '${topicName}' may still exist. Deletion might take some time to propagate."
                    } else {
                        echo "✅ Topic '${topicName}' has been successfully deleted."
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupClientConfig()
            }
        }
        success {
            echo "🎉 Topic deletion pipeline completed successfully!"
        }
        failure {
            echo "💥 Topic deletion pipeline failed. Please check the logs above."
        }
    }
}

def deleteKafkaTopic(topicName) {
    try {
        def deleteOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    kafka-topics --delete \\
                        --topic "${topicName}" \\
                        --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --command-config ${env.CLIENT_CONFIG_FILE}
                '
            """,
            returnStdout: true
        ).trim()

        return "✅ Topic '${topicName}' deletion initiated.\n${deleteOutput}"

    } catch (Exception e) {
        return "ERROR: Failed to delete topic '${topicName}' - ${e.getMessage()}"
    }
}

def verifyTopicExists(topicName) {
    try {
        def listOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    kafka-topics --list \\
                        --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --command-config ${env.CLIENT_CONFIG_FILE}
                ' | grep -x "${topicName}" || echo "NOT_FOUND"
            """,
            returnStdout: true
        ).trim()

        return listOutput != "NOT_FOUND" && listOutput.contains(topicName)

    } catch (Exception e) {
        echo "Warning: Could not verify topic existence - ${e.getMessage()}"
        return false
    }
}

def createKafkaClientConfig(username, password) {
    def securityConfig = ""

    switch(params.SECURITY_PROTOCOL) {
        case 'SASL_PLAINTEXT':
        case 'SASL_SSL':
            securityConfig = """
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            break
        default:
            securityConfig = """
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            break
    }

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
${securityConfig}
EOF'
    """
}

def cleanupClientConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}