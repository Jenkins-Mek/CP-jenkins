properties([
    parameters([
        [$class: 'ChoiceParameter',
            choiceType: 'PT_SINGLE_SELECT',
            name: 'OPERATION',
            description: 'Select a Kafka operation to perform',
            script: [
                $class: 'GroovyScript',
                fallbackScript: [
                    classpath: [],
                    sandbox: true,
                    script: '''return ['ERROR: Unable to load operations']'''
                ],
                script: [
                    classpath: [],
                    sandbox: true,
                    script: '''
                        return [
                            "CREATE_TOPIC",
                            "ALTER_TOPIC",
                            "DELETE_TOPIC",
                            "DESCRIBE_TOPIC",
                            "LIST_TOPICS:selected",
                            "PRODUCER",
                            "CONSUMER",
                            "PRODUCER_SCHEMA",
                            "CONSUMER_SCHEMA",
                            "REGISTER_SCHEMA",
                            "DELETE_SCHEMA",
                            "LIST_SCHEMA"
                        ]
                    '''
                ]
            ]
        ],
        [$class: 'DynamicReferenceParameter',
            choiceType: 'ET_FORMATTED_HTML',
            name: 'TOOL_OPTIONS',
            description: 'Configuration Options for Selected Tool',
            referencedParameters: 'OPERATION',
            script: [
                $class: 'GroovyScript',
                fallbackScript: [
                    classpath: [],
                    sandbox: true,
                    script: '''return ['ERROR: Unable to render tool options']'''
                ],
                script: [
                    classpath: [],
                    sandbox: true,
                    script: '''
                        switch(OPERATION) {
                            case "LIST_TOPICS":
                                return """<div style='padding:10px; background:#e8f5e9; border-left:4px solid #4caf50;'>
                                    <h4>📋 List Topics</h4>
                                    <label><input type='checkbox' name='include_internal' value='true'/> Include internal topics</label>
                                </div>"""
                            case "CREATE_TOPIC":
                                return """<div style='padding:10px; background:#f1f1f1;'>
                                    <h4>🚀 Create Topic</h4>
                                    <input name='topic_name' placeholder='Topic name' type='text'/>
                                    <input name='partitions' placeholder='Partitions (e.g., 3)' type='number'/>
                                    <input name='replication' placeholder='Replication factor (e.g., 1)' type='number'/>
                                </div>"""
                            case "ALTER_TOPIC":
                                return """<div style='padding:10px; background:#f1f1f1;'>
                                    <h4>🛠 Alter Topic</h4>
                                    <input name='topic_name' placeholder='Topic to alter' type='text'/>
                                    <input name='configs' placeholder='Config (e.g., retention.ms=60000)' type='text'/>
                                </div>"""
                            case "DELETE_TOPIC":
                                return """<div style='padding:10px; background:#f8d7da; border-left:4px solid #dc3545;'>
                                    <h4>⚠️ Delete Topic</h4>
                                    <input name='topic_name' placeholder='Topic name to delete' type='text'/>
                                    <label><input type='checkbox' name='confirm' required/> Confirm delete</label>
                                </div>"""
                            case "DESCRIBE_TOPIC":
                                return """<div style='padding:10px; background:#fff3cd; border-left:4px solid #ffc107;'>
                                    <h4>🔍 Describe Topic</h4>
                                    <input name='topic_name' placeholder='Topic name' type='text'/>
                                </div>"""
                            case "PRODUCER":
                            case "CONSUMER":
                                return """<div style='padding:10px; background:#e3f2fd;'>
                                    <h4>${OPERATION == "PRODUCER" ? "📝 Produce" : "📥 Consume"} Messages</h4>
                                    <input name='topic_name' placeholder='Topic name' type='text'/>
                                </div>"""
                            case "PRODUCER_SCHEMA":
                            case "CONSUMER_SCHEMA":
                                return """<div style='padding:10px; background:#ede7f6;'>
                                    <h4>${OPERATION == "PRODUCER_SCHEMA" ? "📝 Produce Schema-based" : "📥 Consume Schema-based"} Messages</h4>
                                    <input name='topic_name' placeholder='Topic name' type='text'/>
                                    <input name='schema_id' placeholder='Schema ID' type='text'/>
                                </div>"""
                            case "REGISTER_SCHEMA":
                                return """<div style='padding:10px; background:#fff;'>
                                    <h4>📦 Register Schema</h4>
                                    <input name='subject' placeholder='Subject (e.g., my-topic-value)' type='text'/>
                                    <textarea name='schema' placeholder='Paste schema JSON here' rows='5' style='width:100%;'></textarea>
                                </div>"""
                            case "DELETE_SCHEMA":
                                return """<div style='padding:10px; background:#ffebee; border-left:4px solid #d32f2f;'>
                                    <h4>🗑 Delete Schema</h4>
                                    <input name='subject' placeholder='Subject name' type='text'/>
                                </div>"""
                            case "LIST_SCHEMA":
                                return """<div style='padding:10px; background:#e8f5e9;'>
                                    <h4>📄 List Registered Schemas</h4>
                                    <p>No additional input required</p>
                                </div>"""
                            default:
                                return "<div><p>Please select a Kafka tool above.</p></div>"
                        }
                    '''
                ]
            ]
        ]
    ])
])


pipeline {
    agent any

    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        CONNECTION_TYPE = 'local-confluent'
        KAFKA_BOOTSTRAP_SERVER = 'localhost:9092'
        SECURITY_PROTOCOL = 'SASL_PLAINTEXT'
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "🚀 Starting Kafka Topic Management"
                    echo "Operation: ${params.OPERATION}"
                }
            }
        }

        stage('Parse Parameters') {
            steps {
                script {
                    def option = "${params.TOPIC_OPTIONS}"
                    def values = option.split(',').collect { it.trim() }.findAll { it }

                    switch(params.OPERATION) {
                        case 'CREATE_TOPIC':
                            env.TOPIC_NAME = values[0]
                            env.PARTITIONS = values[1] ?: '3'
                            env.REPLICATION_FACTOR = values[2] ?: '2'
                            echo "Creating topic: ${env.TOPIC_NAME} (${env.PARTITIONS} partitions, ${env.REPLICATION_FACTOR} replicas)"
                            break
                        case 'DESCRIBE_TOPIC':
                        case 'DELETE_TOPIC':
                            env.TOPIC_NAME = values[0]
                            echo "Topic: ${env.TOPIC_NAME}"
                            break
                        case 'LIST_TOPICS':
                            env.INCLUDE_INTERNAL = values.contains('include_internal') ? 'true' : 'false'
                            echo "Listing all topics (Include internal: ${env.INCLUDE_INTERNAL})"
                            break
                    }
                }
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'CREATE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            if (!env.TOPIC_NAME.matches('^[a-zA-Z0-9._-]+$')) error "Invalid topic name format"
                            echo "✅ Validation passed"
                            break
                        case 'DESCRIBE_TOPIC':
                        case 'DELETE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            echo "✅ Validation passed"
                            break
                        case 'LIST_TOPICS':
                            echo "✅ Validation passed"
                            break
                    }
                }
            }
        }

        stage('Execute Operation') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'LIST_TOPICS':
                            echo "==== Calling List Topic job ===="

                            def envParams = "COMPOSE_DIR=${env.COMPOSE_DIR}," +
                                            "KAFKA_BOOTSTRAP_SERVER=${env.KAFKA_BOOTSTRAP_SERVER}," +
                                            "INCLUDE_INTERNAL=${env.INCLUDE_INTERNAL}," +
                                            "SECURITY_PROTOCOL=${env.SECURITY_PROTOCOL}"

                            build job: 'org-cp-tools/CP-jenkins/list-topics',
                                parameters: [
                                        string(name: 'ParamsAsENV', value: 'true'),
                                        string(name: 'ENVIRONMENT_PARAMS', value: envParams)
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'CREATE_TOPIC':
                            echo "==== Calling Create Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/create-topic',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'PARTITIONS', value: "${env.PARTITIONS}"),
                                    string(name: 'REPLICATION_FACTOR', value: "${env.REPLICATION_FACTOR}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: 'localhost:9092')
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DESCRIBE_TOPIC':
                            echo "==== Calling Describe Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/describe-topic',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DELETE_TOPIC':
                            echo "⚠️ Requesting delete confirmation..."
                            def confirmName = input(
                                message: "Delete topic '${env.TOPIC_NAME}'? This cannot be undone!",
                                parameters: [string(name: 'CONFIRM_NAME', description: "Type topic name to confirm")]
                            )
                            if (confirmName != env.TOPIC_NAME) {
                                error "❌ Confirmation failed - typed '${confirmName}' but expected '${env.TOPIC_NAME}'"
                            }
                            echo "✅ Confirmation successful, proceeding with deletion..."

                            echo "==== Calling Delete Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/delete-topic',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER ?: 'localhost:9092'}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL ?: 'SASL_PLAINTEXT'}"),
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    booleanParam(name: 'CONFIRM_DELETE', value: true)
                                ],
                                propagate: true,
                                wait: true
                            break
                        case 'ALTER_TOPIC':
                            echo "==== Calling Alter Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/alter-topic',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'CONFIGS', value: "${env.CONFIGS}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'PRODUCER':
                            echo "==== Calling Producer job ===="
                            build job: 'org-cp-tools/CP-jenkins/producer',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'CONSUMER':
                            echo "==== Calling Consumer job ===="
                            build job: 'org-cp-tools/CP-jenkins/consumer',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'PRODUCER_SCHEMA':
                            echo "==== Calling Producer Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/producer-schema',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'SCHEMA_ID', value: "${env.SCHEMA_ID}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'CONSUMER_SCHEMA':
                            echo "==== Calling Consumer Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/consumer-schema',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'SCHEMA_ID', value: "${env.SCHEMA_ID}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'REGISTER_SCHEMA':
                            echo "==== Calling Register Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/register-schema',
                                parameters: [
                                    string(name: 'SUBJECT', value: "${env.SUBJECT}"),
                                    text(name: 'SCHEMA', value: "${env.SCHEMA}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DELETE_SCHEMA':
                            echo "==== Calling Delete Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/delete-schema',
                                parameters: [
                                    string(name: 'SUBJECT', value: "${env.SUBJECT}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'LIST_SCHEMA':
                            echo "==== Calling List Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/list-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}")
                                ],
                                propagate: false,
                                wait: true
                            break


                        default:
                            error "❌ Unknown operation: ${params.OPERATION}"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.OPERATION == 'LIST_TOPICS') {
                    archiveArtifacts artifacts: "${env.TOPICS_LIST_FILE}",
                                   fingerprint: true,
                                   allowEmptyArchive: true
                    echo "📦 Topics list archived as artifact"
                }
                echo "✅ Kafka topic operation '${params.OPERATION}' completed successfully"
            }
        }
        failure {
            echo "❌ Kafka topic operation '${params.OPERATION}' failed - check logs for details"
        }
    }
}

