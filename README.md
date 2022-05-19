# PostgreSQL-CDC
Use Debezium to capture Postgres database changes

Here's the full end-to-end setup process

	1. Create a VM (Debian by default) in GCP
  
	2. SSH to the VM, install Docker, Docker compose, Git
  
	3. git clone https://github.com/abedon/PostgreSQL-CDC.git
  
	4. cd PostgreSQL-CDC
  
	5. sudo docker-compose up
  
	6. Once all the containers are up, config postgres container to enable CDC
       Host$ sudo docker exec -it [postgres container ID] bash
             apt-get update && apt-get install postgresql-14-wal2json
             psql -h 127.0.0.1 -U postgres
             psql> alter system set wal_level to 'logical';
             psql> create table test (
                                      id serial primary key,
                                      name varchar
                                     );
       Host$ sudo docker restart [postgres container ID]
  
	7. Host$ echo '
	{
	    "name": "arctype-connector",
	    "config": {
	        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
	        "tasks.max": "1",
	        "plugin.name": "wal2json",
	        "database.hostname": "db",
	        "database.port": "5432",
	        "database.user": "postgres",
	        "database.password": "arctype",
	        "database.dbname": "postgres",
	        "database.server.name": "ARCTYPE",
	        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
	        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
	        "key.converter.schemas.enable": "false",
	        "value.converter.schemas.enable": "false",
	        "snapshot.mode": "always"
	    }
	}
	' > debezium.json
  
	8. Host$ curl -i -X POST \
	         -H "Accept:application/json" \
	         -H "Content-Type:application/json" \
	         127.0.0.1:8083/connectors/ \
	         --data "@debezium.json"
           
	9. Go to Kafka container, check if the topic has been created, it may be created after the CDC message is sent
	/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
  
     In Postgress container, make some changes to the database, e.g.
     insert into test (name) values ('Finally made it!');
  
	10. Watch the CDC topic with live messages (DB changes). Better not try this until the topic is there.
	/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --from-beginning --topic ARCTYPE.public.test --property print.key=true --property key.separator="-"
  
	11. Optionally, on a different machine, install kcat, formerly kafkacat, try the connection to Kafka server from a client outside of the network kafka container and host live in
		a. https://github.com/edenhill/kcat
    
	12. Set up a Dataflow pipeline with Kafka to BigQuery template
		a. Create a BigQuery table with a sample JSON data, so schema will be auto created
		b. Specify BigQuery table / Kafka server / Kafka topic / GCS location for temporary storage
    
	13. Update Postgress database, the CDC data will show up in the BigQuery table
  
	14. To follow up, need to figure out how to restore the CDC data into a materialized BigQuery table instead of a CDC table.
