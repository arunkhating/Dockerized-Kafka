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





🛠 Setup & Deployment Steps

1. Prepare the Environment
2. 
Create your project structure and set permissions to ensure Kafka can read your configuration files.

Bash

mkdir -p deploykafka/kafka-ssl
cd deploykafka

# Ensure the directory is accessible to the Docker user (UID 1000)
chmod -R 755 kafka-ssl

2. Generate SSL Certificates (Self-Signed)
Kafka requires a Truststore and a Keystore. Use these commands to generate a basic CA and a signed certificate for the broker.

Bash
# 1. Generate CA Key and Certificate

openssl req -new -x509 -keyout kafka-ssl/ca-key -out kafka-ssl/ca-cert -days 365 -subj "/CN=MyCompany-CA" -nodes

# 2. Create the Truststore (For Clients & Broker)

keytool -keystore kafka-ssl/kafka.truststore.p12 -alias CARoot -import -file kafka-ssl/ca-cert -storetype PKCS12 -storepass changeit -noprompt

# 3. Create the Keystore (For the Broker)

keytool -keystore kafka-ssl/kafka.keystore.p12 -alias kafka-broker -validity 365 -genkey -keyalg RSA -storetype PKCS12 -storepass changeit -keypass changeit -subj "/CN=10.0.1.118"

# 4. Sign the Broker Certificate with the CA

keytool -keystore kafka-ssl/kafka.keystore.p12 -alias kafka-broker -certreq -file kafka-ssl/cert-file -storepass changeit
openssl x509 -req -CA kafka-ssl/ca-cert -CAkey kafka-ssl/ca-key -in kafka-ssl/cert-file -out kafka-ssl/cert-signed -days 365 -CAcreateserial -passin pass:changeit
keytool -keystore kafka-ssl/kafka.keystore.p12 -alias CARoot -import -file kafka-ssl/ca-cert -storepass changeit -noprompt
keytool -keystore kafka-ssl/kafka.keystore.p12 -alias kafka-broker -import -file kafka-ssl/cert-signed -storepass changeit -noprompt

# 5. Export CA to PEM (For Python/Go Developers)
openssl x509 -in kafka-ssl/ca-cert -out kafka-ssl/ca.pem -outform PEM
3. Create the SASL Configuration
Create the kafka_server_jaas.conf file in your deploykafka folder.

Bash
cat <<EOF > kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    user_admin="admin-secret"
    user_client="client-secret";
};
EOF
4. Deploy the Stack
Run your docker-compose.yml.

Bash
docker-compose up -d
5. Verify the Connection (The "Final Test")
Share these commands in your README so users can verify the SASL_SSL flow immediately after deployment.

Create a Secure Topic:

Bash
docker exec -it kafka kafka-topics --create \
  --bootstrap-server localhost:9095 \
  --replication-factor 1 \
  --partitions 1 \
  --topic secure-test \
  --command-config /etc/kafka/secrets/client-sasl.properties
Produce a Test Message:

Bash
echo "Hello Secure Kafka" | docker exec -i kafka kafka-console-producer \
  --bootstrap-server localhost:9095 \
  --topic secure-test \
  --producer.config /etc/kafka/secrets/client-sasl.properties
Consume the Message:

Bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9095 \
  --topic secure-test \
  --from-beginning \
  --consumer.config /etc/kafka/secrets/client-sasl.properties --max-messages 1
