//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Name of the Kafka topic to create'),
        string(name: 'PARTITIONS', defaultValue: '3', description: 'Number of partitions'),
        string(name: 'REPLICATION_FACTOR', defaultValue: '1', description: 'Replication factor')
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
                        error("❌ TOPIC_NAME parameter is required to create a topic.")
                    }
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

        stage('Create Topic') {
            steps {
                script {
                    def topicName = params.TOPIC_NAME.trim()
                    def partitions = params.PARTITIONS.toInteger()
                    def replicationFactor = params.REPLICATION_FACTOR.toInteger()
                    echo "🆕 Creating Kafka topic: ${topicName} with partitions=${partitions} replicationFactor=${replicationFactor}"
                    def result = createKafkaTopic(topicName, partitions, replicationFactor)
                    echo result
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
    }
}

def createKafkaTopic(topicName, partitions = 3, replicationFactor = 1) {
    try {
        def createOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    kafka-topics --create \\
                        --if-not-exists \\
                        --topic "${topicName}" \\
                        --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --command-config ${env.CLIENT_CONFIG_FILE} \\
                        --partitions ${partitions} \\
                        --replication-factor ${replicationFactor}
                '
            """,
            returnStdout: true
        ).trim()

        return "Topic '${topicName}' created or already exists.\n${createOutput}"
    } catch (Exception e) {
        return "ERROR: Failed to create topic '${topicName}' - ${e.getMessage()}"
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