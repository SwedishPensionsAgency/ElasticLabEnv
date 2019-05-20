# 1)
# Prepare the ES cluster
# start the nodes es01-es03 from the git directory, e.g. '/path/to/ElasticLabEnv'
sudo docker-compose up es0{1..3}

# NOTE! If you get the error:
# ERROR: Get https://docker.elastic.co/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
# You need to create and set proxy information in:
/etc/systemd/system/docker.service.d/https-proxy.conf
# Like so:
[Service]
Environment="HTTP_PROXY=http://myproxy.org:8080/"
Environment="HTTPS_PROXY=http://myproxy.org:8080/"
Environment="NO_PROXY=localhost,127.0.0.1,::1"

# NOTE! If you get this error:
# es01 exited with code 78
# You need to increase vm.max_map_count like so:
sudo sysctl -w vm.max_map_count=262144

# 2)
# add Elasticsearch superuser
# jump into the running es01 container
sudo docker exec -it es01 bash
./bin/elasticsearch-users useradd training -p my_password -r superuser
# ctrl-d to exit es01 container

# 3)
# Start x-pack trial license
curl -XPOST "http://localhost:9200/_xpack/license/start_trial?acknowledge=true"

# 4)
# start Kibana
sudo docker-compose up kibana
# check that you can login to kibana with the user "training"
http://localhost:5602

# 5)
# in Kibana dev console http://localhost:5602/app/kibana#/dev_tools/console?_g=()
# prepare all the roles and users needed by logstash, filebeat, metricbeat
# easiest way is to copy all the lines below and paste them in the dev console and press the green arrow

# Set password for beats_system user used by all *beats
PUT /_security/user/beats_system/_password
{
  "password": "t0p.s3cr3t"
}

#####FILEBEAT####
# Prepare roles for Filebeat
POST /_security/role/filebeat_writer
{
  "cluster": ["manage_index_templates","monitor","manage_ingest_pipelines"],
  "indices": [
    {
      "names": [ "filebeat-*" ],
      "privileges": ["write","create_index"]
    }
  ]
}

POST /_security/role/filebeat_ilm
{
  "cluster": ["manage_ilm"],
  "indices": [
    {
      "names": [ "filebeat-*","shrink-filebeat-*"],
      "privileges": ["write","create_index","manage","manage_ilm"]
    }
  ]
}

# define users used by Filebeat
POST /_security/user/filebeat_internal
{
  "password" : "filebeat-password",
  "roles" : [ "filebeat_writer","kibana_user","filebeat_ilm","beats_system"],
  "full_name" : "Internal Filebeat User"
}

POST /_security/role/filebeat_reader
{
  "indices": [
    {
      "names": [ "filebeat-*" ],
      "privileges": ["read","view_index_metadata"]
    }
  ]
}

POST /_security/user/filebeat_user
{
  "password" : "filebeat-user-password",
  "roles" : [ "filebeat_reader","kibana_user"],
  "full_name" : "Filebeat User"
}

######METRICBEAT#######
# Prepare roles for Metricbeat

POST /_security/role/metricbeat_writer
{
  "cluster": ["manage_index_templates","monitor","manage_ingest_pipelines"],
  "indices": [
    {
      "names": [ "metricbeat-*" ],
      "privileges": ["write","create_index"]
    }
  ]
}

POST /_security/role/metricbeat_ilm
{
  "cluster": ["manage_ilm"],
  "indices": [
    {
      "names": [ "metricbeat-*","shrink-metricbeat-*"],
      "privileges": ["write","create_index","manage","manage_ilm"]
    }
  ]
}

POST /_security/role/metricbeat_reader
{
  "indices": [
    {
      "names": [ "metricbeat-*" ],
      "privileges": ["read","view_index_metadata"]
    }
  ]
}

# prepeare users for metricbeat
POST /_security/user/metricbeat_internal
{
  "password" : "metricbeat-password",
  "roles" : [ "metricbeat_writer","kibana_user","metricbeat_ilm","beats_system"],
  "full_name" : "Internal Metricbeat User"
}

POST /_security/user/metricbeat_user
{
  "password" : "metricbeat-user-password",
  "roles" : [ "metricbeat_reader","kibana_user"],
  "full_name" : "Metricbeat User"
}

#########Logstash#######
# set password for logstash_system user
PUT /_security/user/logstash_system/_password
{
  "password": "t0p.s3cr3t"
}

# prepare roles for logstash
POST /_security/role/logstash_writer
{
  "cluster": ["manage_index_templates", "monitor", "manage_ilm"],
  "indices": [
    {
      "names": [ "logstash-*" ],
      "privileges": ["write","delete","create_index","manage","manage_ilm"]
    }
  ]
}

POST /_security/role/logstash_reader
{
  "indices": [
    {
      "names": [ "logstash-*" ],
      "privileges": ["read","view_index_metadata"]
    }
  ]
}

# prepare users for logstash
POST /_security/user/logstash_internal
{
  "password" : "logstash-password",
  "roles" : [ "logstash_writer"],
  "full_name" : "Internal Logstash User"
}

POST /_security/user/logstash_user
{
  "password" : "x-pack-test-password",
  "roles" : [ "logstash_reader", "logstash_admin"],
  "full_name" : "Kibana User for Logstash"
}

# 6)
# change owner and permissions of metricbeat.yml and filebeat.yml
sudo chown root filebeat.docker.yml metricbeat.yml
sudo chmod go-w filebeat.docker.yml metricbeat.yml

# 7) Setup filebeat and metricbeat index patterns and dashboards in kibana
# Uncomment the 3 line command section for filebeat and metricbeat in docker-compose.yml
# which looks like this

    command:
      - setup
      - --dashboards

# and start filebeat and metricbeat to get them to connect to kibana and run their setups

sudo docker-compose up metricbeat filebeat

# 7)
# comment out the 3 command lines again for both metricbeat and filebeat
# and start them

sudo docker-compose up metricbeat filebeat

#8) start logstash

sudo docker-compose up logstash
