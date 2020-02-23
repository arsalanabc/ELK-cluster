# ELK Cluster: ElasticSearch - Logstash - Kibana
### Run ELK cluster with docker compose

##### First we set up the ElasticSearch container
Add the following lines to your `docker-compose.yml`

    version: '3'
    services:
      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.6.0
        container_name: elasticsearch
        environment:
          - node.name=elasticsearch1
          - cluster.initial_master_nodes=elasticsearch1
          - cluster.name=docker-cluster
          - bootstrap.memory_lock=true
          - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
        ulimits:
          memlock:
            soft: -1
            hard: -1
        volumes:
          - /usr/share/elasticsearch/data
        ports:
          - 9200:9200
The code above will configure a container to run ElasticSearch, binding the ports, attaching the volumes. There is only one node of ES is running.

##### Next set up Kibana container
add the following code to `docker-compose` file.

    kibana:
        image: docker.elastic.co/kibana/kibana:7.6.0
        ports:
          - 5601:5601
          

### Run `docker-compose up`, no need to build since we are pulling the docker images
### Check if cluster is up
Once docker compose is finished, ElasticSearch and Kibana will be running on `localhost` on ports `9200` and `5601` respectively.

###### Create an index with curl
`curl -X PUT "localhost:9200/vehicals?pretty"`

### Elasticsearch APIs
##### Add documents with curl.
#
    curl -X PUT "localhost:9200/vehicals/car/0?pretty" -H 'Content-Type: application/json' -d'{"make":"Honda","year":"2020", "price":10000}'

    curl -X PUT "localhost:9200/vehicals/car/1?pretty" -H 'Content-Type: application/json' -d'{"make":"BMW","year":"2019", "price":70000}'

    curl -X PUT "localhost:9200/vehicals/car/2?pretty" -H 'Content-Type: application/json' -d'{"make":"Lexus","year":"2003", "price":4000}'

This will create 3 documents under `vehicals` index

##### View documents in the index
#
    curl "localhost:9200/vehicals/_search?pretty" #to see all documents
    curl "localhost:9200/vehicals/_doc/1?pretty" #to see document by id = 1
    
##### Next set up Logstash container
Note: We will be using Logstash to push data from a file to ES. We don't neccessarily need a container for this. We can simply run Logstash on the local machine but for consistency we will use a container.

Add the following code to `docker-compose` file.

    logstash:
        image: docker.elastic.co/logstash/logstash:7.6.0
        volumes:
          - ./data/:/usr/share/logstash/pipeline/
          - ./logstash.config:/usr/share/logstash/config/logstash.config
        command:
          logstash -f /usr/share/logstash/config/logstash.config
        depends_on:
          - elasticsearch

We need to bind locat volumes to the container for config files and pipeline for data files. The command `logstash -f /usr/share/logstash/config/logstash.config` is responsible to pushing data to ES so we run it once the Logstash docker is up. ES must be up and running before we run the Logstash command otherwise Logstash won't find ES.
#
Before running the Logstash we need some data files and their config to be uploaded to ES. We will add `cars.csv` in `data`. Create a `logstash.config` and paste the following code in it:

    input {
      file {
        path => "/usr/share/logstash/pipeline/cars.csv"
        start_position => "beginning"
        sincedb_path => "/dev/null"
      }
    }
    
    filter {
      csv {
        separator => ","
        columns => ["manufacturer_name","model_name","transmission","color","odometer_value","year_produced","engine_fuel","engine_has_gas","engine_type","engine_capacity","body_type","has_warranty","state","drivetrain","price_usd","is_exchangeable","location_region","number_of_photos","up_counter","feature_0","feature_1","feature_2","feature_3","feature_4","feature_5","feature_6","feature_7","feature_8","feature_9","duration_listed"]
      }
    }
    
    output {
      elasticsearch {
         hosts => "elasticsearch:9200"
         index => "vehicals"
      }
    stdout {}
    }

Note: path in config file and pipeline volumes in docker-compose must match.

That is it. You have your very first ELK cluster. Ideally running the `docker-compose up` should set up ES, Kibana and run Logstash to push data to ES. Unfortunately, Logstash misbehaves in this tutorial and it is a hit or miss whether Logstash will actually wait for ES to be ready and then push data to it. There are easy ways to fix that but it is out of the scope of this tutorial.
#
We found a work around to it by spinning the Logstash manually once ES and Kibana are up. Do that run `docker-compose up`. Monitor the terminal if Logstash is spun up correctly. If it is then you are good to go. If not, wait for ES container to be ready then spin the Logstash container manually with `docker-compose run logstash` command. Data from `cars.csv` will be added to ES and you can see the data in Kibana.