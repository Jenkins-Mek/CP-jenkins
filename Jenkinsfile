//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Topic name to alter'),

        // Example topic config parameters (add more as needed)
        string(name: 'RETENTION_MS', defaultValue: '', description: 'retention.ms (e.g. 604800000)'),
        string(name: 'CLEANUP_POLICY', defaultValue: '', description: 'cleanup.policy (e.g. delete, compact)'),
        string(name: 'SEGMENT_BYTES', defaultValue: '', description: 'segment.bytes (e.g. 1073741824)'),
        string(name: 'MIN_INSYNC_REPLICAS', defaultValue: '', description: 'min.insync.replicas (e.g. 2)'),
        string(name: 'MAX_MESSAGE_BYTES', defaultValue: '', description: 'max.message.bytes (e.g. 1048576)')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Create Kafka Client Config') {
            steps {
                withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                    script {
                        createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    def hasConfig = [params.RETENTION_MS, params.CLEANUP_POLICY, params.SEGMENT_BYTES, params.MIN_INSYNC_REPLICAS, params.MAX_MESSAGE_BYTES]
                        .any { it?.trim() }
                    if (!hasConfig) {
                        error("❌ At least one config parameter must be set")
                    }
                }
            }
        }

        stage('Build Config Changes') {
            steps {
                script {
                    def configs = [:]

                    if (params.RETENTION_MS?.trim()) configs['retention.ms'] = params.RETENTION_MS.trim()
                    if (params.CLEANUP_POLICY?.trim()) configs['cleanup.policy'] = params.CLEANUP_POLICY.trim()
                    if (params.SEGMENT_BYTES?.trim()) configs['segment.bytes'] = params.SEGMENT_BYTES.trim()
                    if (params.MIN_INSYNC_REPLICAS?.trim()) configs['min.insync.replicas'] = params.MIN_INSYNC_REPLICAS.trim()
                    if (params.MAX_MESSAGE_BYTES?.trim()) configs['max.message.bytes'] = params.MAX_MESSAGE_BYTES.trim()

                    if (configs.isEmpty()) {
                        error("❌ No valid configs to apply")
                    }

                    // Compose comma separated key=value
                    configList = configs.collect { k, v -> "${k}=${v}" }.join(',')
                    echo "Config changes to apply: ${configList}"
                }
            }
        }

        stage('Alter Topic Config') {
            steps {
                script {
                    def cmd = """
                        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                        exec -T broker bash -c '
                            kafka-configs --alter --entity-type topics --entity-name ${params.TOPIC_NAME.trim()} \\
                            --add-config ${configList} \\
                            --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                            --command-config ${CLIENT_CONFIG_FILE}
                        '
                    """

                    echo "Running alter command:\n${cmd}"
                    sh cmd
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
            echo "✅ Topic '${params.TOPIC_NAME}' configuration altered successfully."
        }
        failure {
            echo "❌ Failed to alter topic configuration."
        }
    }
}

def createKafkaClientConfig(username, password) {
    def securityConfig = """
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c 'cat > ${CLIENT_CONFIG_FILE} << EOF
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
${securityConfig}
EOF'
    """
}

def cleanupClientConfig() {
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c "rm -f ${CLIENT_CONFIG_FILE}" || true
    """
}
