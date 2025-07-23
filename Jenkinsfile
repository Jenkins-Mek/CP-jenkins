@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Name of the Kafka topic to create')
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
                        confluentOps.createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Create Topic') {
            steps {
                script {
                    def topicName = params.TOPIC_NAME.trim()
                    echo "🆕 Creating Kafka topic: ${topicName}"
                    confluentOps.createKafkaTopic(topicName)
                    echo "✅ Topic '${topicName}' created successfully."
                }
            }
        }
    }

    post {
        always {
            script {
                confluentOps.cleanupClientConfig()
            }
        }
    }
}
