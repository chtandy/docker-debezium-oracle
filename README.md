### 參考來源
- [Debezium Tutorial](https://github.com/debezium/debezium-examples/tree/main/tutorial)       


### Using Oracle
This assumes Oracle is running on localhost (or is reachable there, e.g. by means of running it within a VM or Docker container with appropriate port configurations) and set up with the configuration, users and grants described in the Debezium Vagrant set-up.
```
# Start the topology as defined in https://debezium.io/documentation/reference/stable/tutorial.html
export DEBEZIUM_VERSION=1.9
docker-compose -f docker-compose-oracle.yaml up --build

# Insert test data
cat debezium-with-oracle-jdbc/init/inventory.sql | docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1
```

The Oracle connector can be used to interact with Oracle either using the Oracle LogMiner API or the Oracle XStreams API.

### LogMiner
The connector by default will use Oracle LogMiner. Adjust the host name of the database server in the register-oracle-logminer.json as per your environment. Then register the Debezium Oracle connector:
```
# Start Oracle connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-oracle-logminer.json

# Create a test change record
echo "INSERT INTO customers VALUES (NULL, 'John', 'Doe', 'john.doe@example.com');" | docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1

# Consume messages from a Debezium topic
docker-compose -f docker-compose-oracle.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic server1.DEBEZIUM.CUSTOMERS

# Modify other records in the database via Oracle client
docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1

# Shut down the cluster
docker-compose -f docker-compose-oracle.yaml down
```


### 用於創建初始快照的默認連接器工作流程
- [Debezium Connector for Oracle](https://debezium.io/documentation/reference/stable/connectors/oracle.html)  
當快照模式設置為默認時，連接器完成以下任務來創建快照：
```
When the snapshot mode is set to the default, the connector completes the following tasks to create a snapshot:

1. Determines the tables to be captured.
2. Obtains a ROW SHARE MODE lock on each of the monitored tables to prevent structural changes from occurring during creation of the snapshot. Debezium holds the locks for only a short time.
3. Reads the current system change number (SCN) position from the server’s redo log.
4. Captures the structure of all relevant tables.
5. Releases the locks obtained in Step 2.
6. Scans all of the relevant database tables and schemas as valid at the SCN position that was read in Step 3 (SELECT * FROM …​ AS OF SCN 123), generates a READ event for each row, and then writes the event records to the table-specific Kafka topic.
7. Records the successful completion of the snapshot in the connector offsets.

## 中翻
1. 確定要捕獲的表。
2. 獲取ROW SHARE MODE每個受監視表的鎖，以防止在創建快照期間發生結構更改。Debezium 持有鎖的時間很短。
3. 從服務器的重做日誌中讀取當前系統更改號 (SCN) 位置。
4. 捕獲所有相關表的結構。
5. 釋放步驟 2 中獲得的鎖。
6. 將所有相關的數據庫表和模式掃描為在步驟 3 ( ) 中讀取的 SCN 位置有效，為每一行SELECT * FROM …​ AS OF SCN 123生成一個READ事件，然後將事件記錄寫入特定於表的 Kafka 主題。
7. 在連接器偏移中記錄快照的成功完成。
```
快照進程開始後，如果進程因連接器故障、重新平衡或其他原因而中斷，連接器重啟後進程會重新啟動。連接器完成初始快照後，它會繼續從其在步驟 3 中讀取的位置進行流式傳輸，以免錯過任何更新。如果連接器由於任何原因再次停止，則在重新啟動後，它將從之前停止的位置恢復流式傳輸更改。





### 使用方式
- clone repo
- modify register-oracle-logminer.json
- docker-compose build
- docker-compose up -d
- `curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-oracle-logminer.json`




