Secure Kafka Deployment (KRaft Mode)This repository contains a production-ready (development-tuned) deployment of Apache Kafka using KRaft (no Zookeeper required).

It features a multi-listener setup providing three distinct access tiers: Plaintext, SSL, and SASL_SSL.

🚀 FeaturesZookeeperless: Uses Kafka KRaft mode for simplified management and faster metadata handling.

Triple-Tier Security:EXTERNAL_SASL: Port 9095 (Encrypted + Authenticated via SASL/PLAIN).INTERNAL: Port 9094 (Encrypted SSL for inter-container communication).EXTERNAL_PLAIN: Port 9092 (Unencrypted for rapid debugging).

Integrated UI: Includes kafka-ui pre-configured to monitor the secure SASL_SSL stream.Automatic SSL: Ready for self-signed or CA-signed certificates.

🛠 PrerequisitesDocker and Docker ComposeOpenSSL (for generating certificates)Python 3.x (if testing with the included client scripts)🚦 Quick 

1. Generate Certificates Ensure your certificates (ca.pem, kafka.truststore.p12, etc.) are placed in the ./kafka-ssl directory.

2. Configure CredentialsEdit the kafka_server_jaas.conf file to manage users:JavaKafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    user_admin="admin-secret"
    user_client="client-secret";
};
3. DeployBashdocker-compose up -d

🔐 Connection Guide for DevelopersPort Mapping SummaryPortProtocolAuthDescription9092PLAINTEXTNoneDevelopment / Debugging only.9094SSLCA CertEncryption only.9095SASL_SSLCA + LoginProduction standard.

Python Client Configuration (confluent-kafka)To connect securely to the HIPAA data topics, use the following configuration:Pythonconf = {
    'bootstrap.servers': 'YOUR_SERVER_IP:9095',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism': 'PLAIN',
    'sasl.username': 'admin',
    'sasl.password': 'admin-secret',
    'ssl.ca.location': 'path/to/ca.pem',
    'ssl.endpoint.identification.algorithm': 'none' # Required for IP-based access
}

🖥 MonitoringAccess the Kafka UI at http://localhost:8080.Default View: Configured to use the admin credentials.

Tip: If you don't see messages, ensure your Isolation Level is set to Read Uncommitted if you are experimenting with transactions.

⚠️ TroubleshootingAPIVERSION_QUERY Error: Ensure you are matching the right protocol to the right port (don't send Plaintext to 9095).

Coordinator Load Error: This often occurs in single-node clusters when Transactions are enabled. 

Set ENABLE_KAFKA_TRANSACTIONS=false for better stability on small deployments.

Message Size: Ensure topic max.message.bytes is not set too low (e.g., avoid the common 19-byte mistake).
