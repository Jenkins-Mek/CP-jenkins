@Library('kafka-ops-shared-lib') _

pipeline {
    agent any
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        CACHE_FILE = 'cached-kafka-topics.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
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
                    // Save to workspace file
                    writeFile file: env.CACHE_FILE, text: topics.join("\n")
                    echo "Kafka topics cached: ${topics}"
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
