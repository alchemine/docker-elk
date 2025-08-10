# Elastic stack (ELK) on Docker
## Original repo: [https://github.com/deviantony/docker-elk](https://github.com/deviantony/docker-elk)

## Customization
1. Add logstash pipelines
    - `general-log`
    - `chat-history`


## Updated features
### elasticsearch
1. Change license type
    - [/elasticsearch/config/elasticsearch.yml](/elasticsearch/config/elasticsearch.yml)
        - Before
            ```yml
            xpack.license.self_generated.type: trial
            ```
        - After
            ```yml
            xpack.license.self_generated.type: basic
            ```


### logstash
1. Add pipelines
    - [/logstash/config/pipeline.yml]
        - Before
            ```yml
            ```
        - After
            ```yml
            # This file is where you define your pipelines. You can define multiple.
            # For more information on multiple pipelines, see the documentation:
            #   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

            - pipeline.id: general-log
            queue.type: persisted
            path.config: pipeline/general-log.conf

            - pipeline.id: chat-history
            queue.type: persisted
            path.config: pipeline/chat-history.conf
            ```
    - [/logstash/pipeline/general-log.conf]
        - Before
            ```conf
            ```
        - After
            ```conf
            input {
                beats {
                    port => 5044
                }

                udp {
                    port => 5959
                    codec => json
                }
                
                tcp {
                    port => 50000
                }
            }

            ## Add your filters / logstash plugins configuration here

            output {
                elasticsearch {
                    hosts => "elasticsearch:9200"
                    user => "logstash_internal"
                    password => "${LOGSTASH_INTERNAL_PASSWORD}"
                    index => "general-log"
                }
            }
            ```
    - [/logstash/pipeline/chat-history.conf]
        - Before
            ```conf
            ```
        - After
            ```conf
            input {
                beats {
                    port => 5045
                }

                udp {
                    port => 5960
                    codec => json
                }
                
                tcp {
                    port => 50001
                }
            }

            ## Add your filters / logstash plugins configuration here

            output {
                elasticsearch {
                    hosts => "elasticsearch:9200"
                    user => "logstash_internal"
                    password => "${LOGSTASH_INTERNAL_PASSWORD}"
                    index => "chat-history"
                }
            }
            ```


### setup
1. Add indices to be handled by logstash
    - [/setup/roles/logstash_writer.json](/setup/roles/logstash_writer.json)
        - Before
            ```json
            {
            "cluster": [
                "manage_index_templates",
                "monitor",
                "manage_ilm"
            ],
            "indices": [
                {
                "names": [
                    "logs-generic-default",
                    "logstash-*",
                    "ecs-logstash-*"
                ],
                "privileges": [
                    "write",
                    "create",
                    "create_index",
                    "manage",
                    "manage_ilm"
                ]
                },
                {
                "names": [
                    "logstash",
                    "ecs-logstash"
                ],
                "privileges": [
                    "write",
                    "manage"
                ]
                }
            ]
            }
            ```
        - After
            ```json
            {
            "cluster": [
                "manage_index_templates",
                "monitor",
                "manage_ilm"
            ],
            "indices": [
                {
                "names": [
                    "logs-generic-default",
                    "logstash-*",
                    "ecs-logstash-*",
                    "general-log",
                    "chat-history"
                ],
                "privileges": [
                    "write",
                    "create",
                    "create_index",
                    "manage",
                    "manage_ilm"
                ]
                },
                {
                "names": [
                    "logstash",
                    "ecs-logstash"
                ],
                "privileges": [
                    "write",
                    "manage"
                ]
                }
            ]
            }
            ```


### docker-compose
1. Expose ports for new pipelines
    - [/docker-compose.yml](/docker-compose.yml)
        - Before
            ```yml
            logstash:
                build:
                context: logstash/
                args:
                    ELASTIC_VERSION: ${ELASTIC_VERSION}
                volumes:
                - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,Z
                - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
                ports:
                - 5044:5044
                - 50000:50000/tcp
                - 50000:50000/udp
                environment:
                LS_JAVA_OPTS: -Xms256m -Xmx256m
                LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
                networks:
                - elk
                depends_on:
                - elasticsearch
                restart: unless-stopped
            ```
        - After
            ```yml
            logstash:
                build:
                context: logstash/
                args:
                    ELASTIC_VERSION: ${ELASTIC_VERSION}
                volumes:
                - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,Z
                - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
                ports:
                - 5044:5044
                - 50000:50000/tcp
                - 5959:5959/udp
                - 5045:5045
                - 50001:50001/tcp
                - 5960:5960/udp
                - 9600:9600
                environment:
                LS_JAVA_OPTS: -Xms256m -Xmx256m
                LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
                networks:
                - elk
                depends_on:
                - elasticsearch
                restart: unless-stopped
            ```
