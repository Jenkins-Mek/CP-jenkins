//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'broker:9092', description: 'Kafka bootstrap server (internal Docker network)'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TEST_TOPIC', defaultValue: 'test-latency-topic', description: 'Topic name for latency testing'),
        string(name: 'NUM_MESSAGES', defaultValue: '100', description: 'Number of messages to send'),
        string(name: 'PRODUCER_THREADS', defaultValue: '1', description: 'Number of producer threads'),
        string(name: 'MESSAGE_SIZE', defaultValue: '512', description: 'Message size in bytes'),
        booleanParam(name: 'CREATE_TOPIC', defaultValue: true, description: 'Create test topic if it does not exist'),
        string(name: 'TOPIC_PARTITIONS', defaultValue: '3', description: 'Number of partitions for test topic (if creating)'),
        string(name: 'TOPIC_REPLICATION_FACTOR', defaultValue: '1', description: 'Replication factor for test topic (if creating)')
    ])
])

pipeline {
    agent any

    environment {
        E2E_RESULTS_FILE = 'kafka-e2e-latency-results.txt'
        CLIENT_CONFIG_FILE = '/etc/kafka/secrets/client.properties'
        TEST_TIMESTAMP = "${new Date().format('yyyy-MM-dd_HH-mm-ss')}"
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    // Validate numeric parameters
                    try {
                        Integer.parseInt(params.NUM_MESSAGES)
                        Integer.parseInt(params.PRODUCER_THREADS)
                        Integer.parseInt(params.MESSAGE_SIZE)
                        Integer.parseInt(params.TOPIC_PARTITIONS)
                        Integer.parseInt(params.TOPIC_REPLICATION_FACTOR)
                    } catch (NumberFormatException e) {
                        error("❌ Invalid numeric parameter: ${e.getMessage()}")
                    }

                    echo "✅ Parameter validation passed"
                    echo "📊 Test Configuration:"
                    echo "  - Topic: ${params.TEST_TOPIC}"
                    echo "  - Messages: ${params.NUM_MESSAGES}"
                    echo "  - Producer Threads: ${params.PRODUCER_THREADS}"
                    echo "  - Message Size: ${params.MESSAGE_SIZE} bytes"
                    echo "  - Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}"
                    echo "  - Security Protocol: ${params.SECURITY_PROTOCOL}"
                }
            }
        }

        stage('Create Test Topic') {
            when {
                expression { params.CREATE_TOPIC }
            }
            steps {
                script {
                    echo "🔨 Creating test topic: ${params.TEST_TOPIC}"
                    
                    try {
                        sh """
                            docker compose --project-directory '${params.COMPOSE_DIR}' \\
                            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                            exec -T broker bash -c '
                                set -e
                                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                                kafka-topics --create \\
                                    --if-not-exists \\
                                    --topic "${params.TEST_TOPIC}" \\
                                    --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                    --command-config ${env.CLIENT_CONFIG_FILE} \\
                                    --partitions ${params.TOPIC_PARTITIONS} \\
                                    --replication-factor ${params.TOPIC_REPLICATION_FACTOR}
                            '
                        """
                        echo "✅ Test topic created/verified successfully"
                    } catch (Exception e) {
                        echo "⚠️ Topic creation failed, continuing with existing topic: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Verify Topic Exists') {
            steps {
                script {
                    echo "🔍 Verifying test topic exists: ${params.TEST_TOPIC}"
                    
                    def topics = sh(
                        script: """
                            docker compose --project-directory '${params.COMPOSE_DIR}' \\
                            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                            exec -T broker bash -c '
                                set -e
                                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                                kafka-topics --list \\
                                    --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                    --command-config ${env.CLIENT_CONFIG_FILE}
                            ' 2>/dev/null
                        """,
                        returnStdout: true
                    ).trim().split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') && !it.contains('FATAL') }
                    if (!topics.contains(params.TEST_TOPIC)) {
                        error("❌ Test topic '${params.TEST_TOPIC}' does not exist. Enable 'CREATE_TOPIC' or create the topic manually.")
                    }
                    
                    echo "✅ Test topic verified: ${params.TEST_TOPIC}"
                    
                    // Describe the topic
                    def description = sh(
                        script: """
                            docker compose --project-directory '${params.COMPOSE_DIR}' \\
                            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                            exec -T broker bash -c '
                                set -e
                                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                                kafka-topics --describe \\
                                    --topic "${params.TEST_TOPIC}" \\
                                    --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                    --command-config ${env.CLIENT_CONFIG_FILE}
                            ' 2>/dev/null
                        """,
                        returnStdout: true
                    ).trim()
                    echo "📋 Topic configuration:\n${description}"
                }
            }
        }

        stage('Run E2E Latency Test') {
            steps {
                script {
                    echo "🚀 Starting Kafka E2E latency test..."
                    echo "⏱️ Test started at: ${new Date()}"
                    
                    def testResults = sh(
                        script: """
                            docker compose --project-directory '${params.COMPOSE_DIR}' \\
                            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                            exec -T broker bash -c '
                                set -e
                                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                                echo "Starting E2E latency test..."
                                kafka-e2e-latency \\
                                    ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                    "${params.TEST_TOPIC}" \\
                                    ${params.NUM_MESSAGES} \\
                                    ${params.PRODUCER_THREADS} \\
                                    ${params.MESSAGE_SIZE} \\
                                    ${env.CLIENT_CONFIG_FILE}
                            ' 2>&1
                        """,
                        returnStdout: true
                    ).trim()

                    echo "📊 E2E Latency Test Results:"
                    echo testResults

                    // Parse and highlight key metrics
                    def lines = testResults.split('\n')
                    def avgLatency = lines.find { it.contains('Avg latency') || it.contains('Average latency') }
                    def p99Latency = lines.find { it.contains('99th percentile') || it.contains('p99') }
                    def throughput = lines.find { it.contains('throughput') || it.contains('Throughput') }

                    if (avgLatency) echo "🎯 ${avgLatency}"
                    if (p99Latency) echo "📈 ${p99Latency}"
                    if (throughput) echo "🔄 ${throughput}"

                    // Save detailed results
                    saveTestResults(testResults)
                }
            }
        }

        stage('Analyze Results') {
            steps {
                script {
                    echo "📈 Analyzing test results..."
                    
                    def resultsFile = readFile(env.E2E_RESULTS_FILE)
                    def lines = resultsFile.split('\n')
                    
                    // Extract key metrics (this will depend on the actual output format)
                    def avgLatencyLine = lines.find { it.toLowerCase().contains('avg') && it.toLowerCase().contains('latency') }
                    def maxLatencyLine = lines.find { it.toLowerCase().contains('max') && it.toLowerCase().contains('latency') }
                    
                    if (avgLatencyLine || maxLatencyLine) {
                        echo "✅ Test completed successfully!"
                        echo "📋 Key Metrics Summary:"
                        if (avgLatencyLine) echo "  - ${avgLatencyLine}"
                        if (maxLatencyLine) echo "  - ${maxLatencyLine}"
                    } else {
                        echo "⚠️ Could not parse latency metrics from output"
                    }
                    
                    echo "💾 Full results saved to artifact: ${env.E2E_RESULTS_FILE}"
                }
            }
        }
    }

    post {
        always {
            script {
                echo "🧹 Cleaning up test environment..."
            }
        }
        success {
            script {
                archiveArtifacts artifacts: "${env.E2E_RESULTS_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Test results archived successfully: ${env.E2E_RESULTS_FILE}"
                echo "✅ E2E Latency Test completed successfully!"
                
                // Optional: Clean up test topic
                if (params.CREATE_TOPIC && env.CLEANUP_TEST_TOPIC == 'true') {
                    try {
                        sh """
                            docker compose --project-directory '${params.COMPOSE_DIR}' \\
                            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                            exec -T broker bash -c '
                                set -e
                                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                                kafka-topics --delete \\
                                    --topic "${params.TEST_TOPIC}" \\
                                    --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                    --command-config ${env.CLIENT_CONFIG_FILE}
                            '
                        """
                        echo "🗑️ Test topic cleaned up: ${params.TEST_TOPIC}"
                    } catch (Exception e) {
                        echo "⚠️ Failed to cleanup test topic: ${e.getMessage()}"
                    }
                }
            }
        }
        failure {
            script {
                echo "❌ E2E Latency Test failed!"
                echo "📋 Check the console output for detailed error information"
                
                // Try to save partial results if any
                try {
                    def errorInfo = """# Kafka E2E Latency Test - FAILED
# Test failed at: ${new Date()}
# Configuration:
#   Topic: ${params.TEST_TOPIC}
#   Messages: ${params.NUM_MESSAGES}
#   Producer Threads: ${params.PRODUCER_THREADS}
#   Message Size: ${params.MESSAGE_SIZE}
#   Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}

ERROR: Test execution failed. Check Jenkins console output for details.
"""
                    writeFile file: env.E2E_RESULTS_FILE, text: errorInfo
                    archiveArtifacts artifacts: "${env.E2E_RESULTS_FILE}", fingerprint: true, allowEmptyArchive: true
                } catch (Exception e) {
                    echo "Failed to create error artifact: ${e.getMessage()}"
                }
            }
        }
    }
}

// Helper function to list Kafka topics
def listKafkaTopics() {
    def topicsOutput = sh(
        script: """
            docker compose --project-directory '${params.COMPOSE_DIR}' \\
            -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
            exec -T broker bash -c '
                set -e
                unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                kafka-topics --list \\
                    --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                    --command-config ${env.CLIENT_CONFIG_FILE}
            ' 2>/dev/null
        """,
        returnStdout: true
    ).trim()

    def allTopics = topicsOutput.split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') && !it.contains('FATAL') }
    return allTopics.findAll { !it.startsWith('_') } // Filter out internal topics
}

// Helper function to describe a Kafka topic
def describeKafkaTopic(topicName) {
    try {
        def describeOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' \\
                -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    kafka-topics --describe \\
                        --topic "${topicName}" \\
                        --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --command-config ${env.CLIENT_CONFIG_FILE}
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        return describeOutput
    } catch (Exception e) {
        return "ERROR: Failed to describe topic '${topicName}' - ${e.getMessage()}"
    }
}

// Helper function to save test results
def saveTestResults(testOutput) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def testConfig = """# Kafka E2E Latency Test Results
# Generated: ${timestamp}
# Test ID: ${env.TEST_TIMESTAMP}
# Configuration:
#   Topic: ${params.TEST_TOPIC}
#   Messages: ${params.NUM_MESSAGES}
#   Producer Threads: ${params.PRODUCER_THREADS}
#   Message Size: ${params.MESSAGE_SIZE} bytes
#   Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
#   Security Protocol: ${params.SECURITY_PROTOCOL}
#   Topic Partitions: ${params.TOPIC_PARTITIONS}
#   Replication Factor: ${params.TOPIC_REPLICATION_FACTOR}

================================================================================
E2E LATENCY TEST OUTPUT
================================================================================
${testOutput}

================================================================================
END OF RESULTS
================================================================================
"""

    writeFile file: env.E2E_RESULTS_FILE, text: testConfig
}