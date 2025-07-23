@Library('kafka-ops-shared-lib') _
pipeline {
    agent any
    parameters {
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory')
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server')
        string(name: 'SECURITY_PROTOCOL', defaultValue: 'SASL_PLAINTEXT', description: 'Security protocol')
    }
    environment {
        CACHE_FILE = 'cached-kafka-topics.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }
    stages {
        stage('Debug Parameters') {
            steps {
                script {
                    echo "COMPOSE_DIR param: ${params.COMPOSE_DIR}"
                    echo "KAFKA_BOOTSTRAP_SERVER param: ${params.KAFKA_BOOTSTRAP_SERVER}"
                    echo "SECURITY_PROTOCOL param: ${params.SECURITY_PROTOCOL}"
                    echo "CLIENT_CONFIG_FILE env: ${env.CLIENT_CONFIG_FILE}"
                    // Verify the compose file exists
                    sh "ls -la ${params.COMPOSE_DIR}/docker-compose.yml || echo 'Compose file not found'"
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
        stage('Fetch Kafka Topics') {
            steps {
                script {
                    def topics = confluentOps.listKafkaTopics()
                    if (topics.isEmpty()) {
                        error("No topics found!")
                    }
                    writeFile file: env.CACHE_FILE, text: topics.join("\n")
                    echo "Kafka topics cached to file: ${env.CACHE_FILE}"
                }
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: "${env.CACHE_FILE}", fingerprint: true
        }
        always {
            script {
                confluentOps.cleanupClientConfig()
            }
        }
    }
}