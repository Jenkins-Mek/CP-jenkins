//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], description: 'Security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name'),
        choice(name: 'INPUT_MODE', choices: ['Simple', 'Advanced'], description: 'Choose input method'),


        string(name: 'RETENTION_DAYS', defaultValue: '', description: 'Retention period (in days)'),
        choice(name: 'CLEANUP_POLICY', choices: ['delete', 'compact', 'delete,compact'], description: 'Kafka cleanup.policy'),
        string(name: 'SEGMENT_BYTES', defaultValue: '', description: 'Segment size in bytes (e.g. 1073741824)'),
        string(name: 'MIN_INSYNC_REPLICAS', defaultValue: '', description: 'Minimum in-sync replicas'),
        string(name: 'MAX_MESSAGE_BYTES', defaultValue: '', description: 'Maximum message size in bytes'),

        text(name: 'RAW_CONFIGS', defaultValue: '', description: 'Advanced mode: key=value,key=value (e.g. retention.ms=60000,cleanup.policy=compact)')
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

                    def hasConfig = [
                        params.RETENTION_DAYS,
                        params.CLEANUP_POLICY,
                        params.SEGMENT_BYTES,
                        params.MIN_INSYNC_REPLICAS,
                        params.MAX_MESSAGE_BYTES,
                        params.RAW_CONFIGS
                    ].any { it?.trim() }

                    if (!hasConfig) {
                        error("❌ At least one configuration must be provided.")
                    }
                }
            }
        }

        stage('Build Config Changes') {
            steps {
                script {
                    def escapeConfigValue = { val ->
                        val.contains(',') ? val.replace(',', '\\,') : val
                    }

                    if (params.INPUT_MODE == 'Advanced') {
                        if (!params.RAW_CONFIGS?.trim()) {
                            error("❌ RAW_CONFIGS cannot be empty in Advanced mode.")
                        }
                        configList = params.RAW_CONFIGS.trim()
                    } else {
                        def configs = [:]

                        if (params.RETENTION_DAYS?.trim()) {
                            def days = params.RETENTION_DAYS.trim().toInteger()
                            configs['retention.ms'] = (days * 24 * 60 * 60 * 1000).toString()
                        }
                        if (params.CLEANUP_POLICY?.trim()) {
                            configs['cleanup.policy'] = params.CLEANUP_POLICY.trim()
                        }
                        if (params.SEGMENT_BYTES?.trim()) {
                            configs['segment.bytes'] = params.SEGMENT_BYTES.trim()
                        }
                        if (params.MIN_INSYNC_REPLICAS?.trim()) {
                            configs['min.insync.replicas'] = params.MIN_INSYNC_REPLICAS.trim()
                        }
                        if (params.MAX_MESSAGE_BYTES?.trim()) {
                            configs['max.message.bytes'] = params.MAX_MESSAGE_BYTES.trim()
                        }

                        if (configs.isEmpty()) {
                            error("❌ No valid configs to apply in Simple mode.")
                        }

                        configList = configs.collect { k, v ->
                            def escapedValue = v.contains(',') ? "[${v}]" : v
                            "${k}=${escapedValue}"
                        }.join(',')
                    }

                    echo "🛠 Config changes to apply: ${configList}"
                }
            }
        }

        stage('Alter Topic Config') {
            steps {
                script {
                    def cmd = """
                        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                        exec -T broker bash -c '
                            set -e
                            unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                            kafka-configs --alter --entity-type topics --entity-name ${params.TOPIC_NAME.trim()} \\
                            --add-config "${configList}" \\
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
