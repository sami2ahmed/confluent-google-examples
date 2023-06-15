# dataflow-confluentcloud
Using GCP Dataflow and Confluent Cloud to create a raffle system 

# Prerequisities (both are free)
1. Google Cloud Platform (GCP) account 
2. Confluent Cloud (CC) account 

## Confluent Cloud 
1. All these steps can be carried out in the CC dashboard. First, create a Basic cluster, prefably in a region near where your GCP resources (Bigquery) will live. Select GCP, Region of `us-east4`, and `single zone` availability. Give `cluster name` whatever you would like. Hit `launch cluster.`
2. In `Topics` menu of the dashboard, Create a 1-partition topic called `application.` All other defaults are fine.
3. In `Cluster Overview`, select `API Keys` and `create key.` Create a `Global Access` API key and secret, download them for later use. 

## GCP 
1. Login to your GCP console. Within Bigquery, create two datasets: `promotions` and `raffle_dataset` with multi-region set to `US`. No other settings are required. With `promotions` run this sql to create necessary fields: 
```
CREATE TABLE <your project>.promotions.dailygiftcardwinners (
  day DATE,
  winnernumber INTEGER,
  promocode STRING,
)
```

2. Launch a Cloud shell terminal and authenticate `gcloud auth login` make sure you are using your desired project `gcloud config set project <project-name>`
3. git clone this repo using the Cloud shell i.e. `https://github.com/confluentinc/confluent-google-examples` and then `cd confluent-google-examples/ccloud-dataflow-demo`
4. Configure your `beam-demo.config` in the repo's entry-producer directory with your Confluent Cloud connection details, example below:

bootstrap server can be found in `cluster settings` of the CC dashboard. The sasl username/password is your api key/secret from earlier.
``` 
bootstrap.servers=pkc-xxxxx.us-east4.gcp.confluent.cloud:9092 
sasl.mechanisms=PLAIN
sasl.username=
sasl.password=
```

5. Within the `DataflowPipeline.java` in the `entry-df-pipeline` directory, you need to input all instances of `<your-bootstrap-server>` from your CC cluster (example:`pkc-xy2aa.us-east-2.gcp.confluent.cloud:9092`)

6. Go to key management in GCP console and create a new key ring. set region to `us-east4`. 
7. Create a keyname of `cflt-usr` with software protection level. Generated key for key material. Symmetric encrypt/decrypt. default key rotation is fine as this is for demo purposes only. Set destruction to a high value like 120. 
8. Do the same for keyname of `cflt-pwd`. 
10. Navigate to Cloud Storage in the console and create `stg` and `temp` buckets. Both in `us-east4`. 

### setup a SA to be used for access token 
1. in GCP console, Navigate to Service accounts in the GCP console and create a Service account. Make it a role of owner and name it whatever you would like. Grant yourself service account users role and admins role. Go to Keys > add key > JSON key and download it. 

### example curl commands below to do encryption process. You will need to replace with your own unique paths. I like to do this locally for some reason, sure could be done in gcloud shell  
1. gcloud auth login
2. gcloud config set project <project-name>
3. set GOOGLE_APPLICATION_CREDENTIALS=<path-to-creds>
4. export accesstoken=$(gcloud auth application-default print-access-token)
6. base 64 encode apikey/secret from CC, and save the output to a local file: 

Example, you will need to replace with your CC creds that ypu downloaded earlier:
`echo -n '<>' | base64`
`echo -n '<>' | base64`

Then feed the base64 encoded key/secret into the data (after plaintext in the <>) fields of these commands. You will also need to configure your project, location, keyring, and key usr/pwd accordingly: 

### encrypt your CC key with base64 cred, replace all <>
```
curl "https://cloudkms.googleapis.com/v1/projects/<>/locations/<>/keyRings/<>/cryptoKeys/<>:encrypt" \
--request "POST" \
--header "authorization: Bearer $accesstoken" \
--header "content-type: application/json" \
--data "{\"plaintext\": \"<>\"}"
```

### encrypt CC pwd with base64 cred, replace all <>
```
curl "https://cloudkms.googleapis.com/v1/projects/<>/locations/<>/keyRings/<>/cryptoKeys/<>:encrypt" \
--request "POST" \
--header "authorization: Bearer $accesstoken" \
--header "content-type: application/json" \
--data "{\"plaintext\": \"<>\"}"
```

7. save the `cipertext` outputted by the encryption process so we can use it in our dataflow process later on. 

# Run Producer
1. In GCP Cloud shell, cd into the `entry-producer` dir and `pip install -r requirements.txt` 
2. Create a data output file i.e. `touch data.out` 
3. Kickoff your producer with a large number of entries. Note your dataflow will not run if the producer stops emitting records: `python producer.py -f beam-demo.config -t entries -m 4000`. You can simply restart the producer with the same command (nondestructive action) to emit more entries. 

# Run Dataflow 
1. In a new GCP cloud shell, set java version to 11 via `sudo update-alternatives --config java` and select option `1`. 
2. cd into the `entry-df-pipeline` dir, and compile the project `mvn package` 
3. check if the producer is still running, if not kick off more entries, then run dataflow:

```
java -jar target/ccloud-dataflow-demo-1.0-SNAPSHOT.jar --runner=DataflowRunner --project=<> --region=us-east4 --keyRing=<> --stagingLocation=gs://<>/stg --tempLocation=gs://<>/temp --kmsUsernameKeyId=cflt-usr --confluentCloudEncryptedUsername=<> --kmsPasswordKeyId=cflt-pwd --confluentCloudEncryptedPassword=<>
```

You should see data flowing into final topic called application in CC by the end, this might take a few moments 








