Instruction to reproduce pipeline
==================================
General
-----------------------------------
Nifi pipeline that does next:
- Ingest stream with Nifi
- split (and transform data if needed)
- assign attribute to each batch of data (of name "batchid" that is timestamp of ingestion of this portion of data)
- write earch batch to Kafka
- but consume with CLI consumer

All services need to be run whithin Docker container.

> Nifi pipeline: general view.

 ![Pipeline](https://raw.githubusercontent.com/AnnieKey/NifiKafkaStreaming/master/screenshouts/schema.png)
 
 Сustomize containers.
----------------------------------
Create docker-compose.yml with next content:
*version: '2'
services:*

  *NiFi:
    image: apache/nifi:latest
    restart: on-failure
    ports:
    - "8080:8080"
    depends_on:
      - kafka*

 *zookeeper:
    image: wurstmeister/zookeeper:latest
    expose:
    - "2181"*

 *kafka:
    image: wurstmeister/kafka:latest
    depends_on:
    - zookeeper
    ports:
    - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://docker_kafka_1:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: docker_zookeeper_1:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT*

It will run 3 containers with zookeeper, kafka and nifi.

Command for check: **docker ps**.
Result:
*CONTAINER_ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                         NAMES
7588f1a9686d        apache/nifi:latest              "../scripts/start.sh"    14 hours ago        Up 14 hours         8443/tcp, 0.0.0.0:8080->8080/tcp, 10000/tcp   docker_NiFi_1
9b12706a1539        wurstmeister/kafka:latest       "start-kafka.sh"         14 hours ago        Up 14 hours         0.0.0.0:9092->9092/tcp                        docker_kafka_1
452f2a3eb572        wurstmeister/zookeeper:latest   "/bin/sh -c '/usr/sb…"   14 hours ago        Up 14 hours         22/tcp, 2181/tcp, 2888/tcp, 3888/tcp          docker_zookeeper_1*

Commands to customaze kafka broker:
1. docker exec -it 9b12706a1539 bash
> 9b12706a1539 - kafka container's id
2. cd opt/kafka_2.12-2.3.0/
3. bin/kafka-topics.sh --bootstrap-server docker_kafka_1:9092 --create --topic tweets --partitions 5 --replication-factor 1
> create topic with name tweets
4. bin/kafka-topics.sh --bootstrap-server docker_kafka_1:9092 --list
> show the list of all topics
5. bin/kafka-topics.sh --bootstrap-server docker_kafka_1:9092 --describe --topic tweets
> show all info about topic tweets
6. bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list docker_kafka_1:9092 --topic tweets --time -1 --offsets 1 | awk -F ":" '{sum += $3} END {print sum}'
> show the number of messages in topic tweets
> now it shows 0
> after reading data from Nifi -- 5030 (or another number, depend on how many messages have you send)
7. bin/kafka-console-consumer.sh  --bootstrap-server docker_kafka_1:9092 --topic tweets --from-beginning
> read messages:
> Example: [{"created_at":"Fri Aug 23 07:15:11 +0000 2019","id":1164798222419632129,"source":"<a href=\"http://twitter.com/download/android\" rel=\"nofollow\">Twitter for Android</a>","user":"MapRecord[{utc_offset=null, friends_count=369, profile_image_url_https=https://pbs.twimg.com/profile_images/1149074498051817472/5XH6qtnJ_normal.jpg, listed_count=0, profile_background_image_url=, default_profile_image=false, favourites_count=10546, description=There is no end though there is a start in space. — Infinity., created_at=Fri Jun 23 14:16:28 +0000 2017, is_translator=false, profile_background_image_url_https=, protected=false, screen_name=JDou19, id_str=878255438218883072, profile_link_color=1DA1F2, translator_type=none, id=878255438218883072, geo_enabled=false, profile_background_color=F5F8FA, lang=null, profile_sidebar_border_color=C0DEED, profile_text_color=333333, verified=false, profile_image_url=http://pbs.twimg.com/profile_images/1149074498051817472/5XH6qtnJ_normal.jpg, time_zone=null, url=null, contributors_enabled=false, profile_background_tile=false, profile_banner_url=https://pbs.twimg.com/profile_banners/878255438218883072/1557162780, statuses_count=1612, follow_request_sent=null, followers_count=36, profile_use_background_image=true, default_profile=true, following=null, name=Juan, location=null, profile_sidebar_fill_color=DDEEF6, notifications=null}]","geo":null,"coordinates":null,"place":null,"quote_count":"0","reply_count":"0","retweet_count":"0","favourite_count":null,"timestamp_ms":"1566544511786"}]



Сustomize Nifi processors.
----------------------------------

### GetTwitter processor.
From official documentation: Pulls status changes from Twitter's streaming API.
> Add credentials (keys and tokens) for twitter app;
> generate ID -> https://tweeterid.com/ ;
> set filters.

![GetTwitter](https://raw.githubusercontent.com/AnnieKey/NifiKafkaStreaming/master/screenshouts/getTwitter.png)

### PublishKafka processor.
From official documentation: Sends the contents of a FlowFile as a message to Apache Kafka using the Kafka 0.9.x Producer. The messages to send may be individual FlowFiles or may be delimited, using a user-specified delimiter, such as a new-line. Please note there are cases where the publisher can get into an indefinite stuck state. We are closely monitoring how this evolves in the Kafka community and will take advantage of those fixes as soon as we can. In the mean time it is possible to enter states where the only resolution will be to restart the JVM NiFi runs on. The complementary NiFi processor for fetching messages is ConsumeKafka.
> Add Kafka broker and topic name.

![PublishKafka](https://raw.githubusercontent.com/AnnieKey/NifiKafkaStreaming/master/screenshouts/publishKafka.png)

### UpdateAttribute processor.
From official documentation:Updates the Attributes for a FlowFile by using the Attribute Expression Language and/or deletes the attributes based on a regular expression.

> Add timestamp attribute.

![UpdateAttribute](https://raw.githubusercontent.com/AnnieKey/NifiKafkaStreaming/master/screenshouts/timestamp_customize.png)

> see the result

![UpdateAttribute](https://raw.githubusercontent.com/AnnieKey/NifiKafkaStreaming/master/screenshouts/timestamp_result.png)

_______________________________________________________________________________________________________________

Results
---------------------

### Input tweets

*{
  "created_at" : "Fri Aug 23 07:54:57 +0000 2019",
  "id" : 1164808230649733122,
  "id_str" : "1164808230649733122",
  "text" : "RT @_Human_Thought_: marvel fans when they see spider-man  is leaving the MCU because of sony and disney #SpiderMan https://t.co/pCzeb7oEnV",
  "source" : "<a href=\"http://twitter.com/download/iphone\" rel=\"nofollow\">Twitter for iPhone</a>",
  "truncated" : false,
  "in_reply_to_status_id" : null,
  "in_reply_to_status_id_str" : null,
  "in_reply_to_user_id" : null,
  "in_reply_to_user_id_str" : null,
  "in_reply_to_screen_name" : null,
  "user" : {
    "id" : 1004024363572760576,
    "id_str" : "1004024363572760576",
    "name" : "Balsita🐼🇺🇾",
    "screen_name" : "_PandaMalik",
    "location" : "You’ll Float Too🎈",
    "url" : null,
    "description" : "Professional fangirl.",
    "translator_type" : "none",
    "protected" : false,
    "verified" : false,
    "followers_count" : 114,
    "friends_count" : 375,
    "listed_count" : 0,
    "favourites_count" : 10407,
    "statuses_count" : 1403,
    "created_at" : "Tue Jun 05 15:37:16 +0000 2018",
    "utc_offset" : null,
    "time_zone" : null,
    "geo_enabled" : false,
    "lang" : null,
    "contributors_enabled" : false,
    "is_translator" : false,
    "profile_background_color" : "F5F8FA",
    "profile_background_image_url" : "",
    "profile_background_image_url_https" : "",
    "profile_background_tile" : false,
    "profile_link_color" : "1DA1F2",
    "profile_sidebar_border_color" : "C0DEED",
    "profile_sidebar_fill_color" : "DDEEF6",
    "profile_text_color" : "333333",
    "profile_use_background_image" : true,
    "profile_image_url" : "http://pbs.twimg.com/profile_images/1164689523437178880/Vy4GbaK__normal.jpg",
    "profile_image_url_https" : "https://pbs.twimg.com/profile_images/1164689523437178880/Vy4GbaK__normal.jpg",
    "profile_banner_url" : "https://pbs.twimg.com/profile_banners/1004024363572760576/1535392622",
    "default_profile" : true,
    "default_profile_image" : false,
    "following" : null,
    "follow_request_sent" : null,
    "notifications" : null
  },
  "geo" : null,
  "coordinates" : null,
  "place" : null,
  "contributors" : null,
  "retweeted_status" : {
    "created_at" : "Tue Aug 20 23:15:37 +0000 2019",
    "id" : 1163952756845142016,
    "id_str" : "1163952756845142016",
    "text" : "marvel fans when they see spider-man  is leaving the MCU because of sony and disney #SpiderMan https://t.co/pCzeb7oEnV",
    "display_text_range" : [ 0, 94 ],
    "source" : "<a href=\"http://twitter.com/download/android\" rel=\"nofollow\">Twitter for Android</a>",
    "truncated" : false,
    "in_reply_to_status_id" : null,
    "in_reply_to_status_id_str" : null,
    "in_reply_to_user_id" : null,
    "in_reply_to_user_id_str" : null,
    "in_reply_to_screen_name" : null,
    "user" : {
      "id" : 939609548536545280,
      "id_str" : "939609548536545280",
      "name" : "Gorzki_Aloes",
      "screen_name" : "_Human_Thought_",
      "location" : "Poznań, Polska",
      "url" : null,
      "description" : "Jezus nie istnieje, a partia jest na zawsze",
      "translator_type" : "none",
      "protected" : false,
      "verified" : false,
      "followers_count" : 21,
      "friends_count" : 214,
      "listed_count" : 0,
      "favourites_count" : 7399,
      "statuses_count" : 1844,
      "created_at" : "Sat Dec 09 21:35:48 +0000 2017",
      "utc_offset" : null,
      "time_zone" : null,
      "geo_enabled" : false,
      "lang" : null,
      "contributors_enabled" : false,
      "is_translator" : false,
      "profile_background_color" : "000000",
      "profile_background_image_url" : "http://abs.twimg.com/images/themes/theme1/bg.png",
      "profile_background_image_url_https" : "https://abs.twimg.com/images/themes/theme1/bg.png",
      "profile_background_tile" : false,
      "profile_link_color" : "19CF86",
      "profile_sidebar_border_color" : "000000",
      "profile_sidebar_fill_color" : "000000",
      "profile_text_color" : "000000",
      "profile_use_background_image" : false,
      "profile_image_url" : "http://pbs.twimg.com/profile_images/1033411884903485441/64C6KpAO_normal.jpg",
      "profile_image_url_https" : "https://pbs.twimg.com/profile_images/1033411884903485441/64C6KpAO_normal.jpg",
      "profile_banner_url" : "https://pbs.twimg.com/profile_banners/939609548536545280/1563220127",
      "default_profile" : false,
      "default_profile_image" : false,
      "following" : null,
      "follow_request_sent" : null,
      "notifications" : null
    },
    "geo" : null,
    "coordinates" : null,
    "place" : null,
    "contributors" : null,
    "is_quote_status" : false,
    "quote_count" : 7,
    "reply_count" : 3,
    "retweet_count" : 272,
    "favorite_count" : 675,
    "entities" : {
      "hashtags" : [ {
        "text" : "SpiderMan",
        "indices" : [ 84, 94 ]
      } ],
      "urls" : [ ],
      "user_mentions" : [ ],
      "symbols" : [ ],
      "media" : [ {
        "id" : 1163952755037429767,
        "id_str" : "1163952755037429767",
        "indices" : [ 95, 118 ],
        "media_url" : "http://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
        "media_url_https" : "https://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
        "url" : "https://t.co/pCzeb7oEnV",
        "display_url" : "pic.twitter.com/pCzeb7oEnV",
        "expanded_url" : "https://twitter.com/_Human_Thought_/status/1163952756845142016/photo/1",
        "type" : "photo",
        "sizes" : {
          "thumb" : {
            "w" : 150,
            "h" : 150,
            "resize" : "crop"
          },
          "medium" : {
            "w" : 1024,
            "h" : 536,
            "resize" : "fit"
          },
          "small" : {
            "w" : 680,
            "h" : 356,
            "resize" : "fit"
          },
          "large" : {
            "w" : 1024,
            "h" : 536,
            "resize" : "fit"
          }
        }
      } ]
    },
    "extended_entities" : {
      "media" : [ {
        "id" : 1163952755037429767,
        "id_str" : "1163952755037429767",
        "indices" : [ 95, 118 ],
        "media_url" : "http://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
        "media_url_https" : "https://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
        "url" : "https://t.co/pCzeb7oEnV",
        "display_url" : "pic.twitter.com/pCzeb7oEnV",
        "expanded_url" : "https://twitter.com/_Human_Thought_/status/1163952756845142016/photo/1",
        "type" : "photo",
        "sizes" : {
          "thumb" : {
            "w" : 150,
            "h" : 150,
            "resize" : "crop"
          },
          "medium" : {
            "w" : 1024,
            "h" : 536,
            "resize" : "fit"
          },
          "small" : {
            "w" : 680,
            "h" : 356,
            "resize" : "fit"
          },
          "large" : {
            "w" : 1024,
            "h" : 536,
            "resize" : "fit"
          }
        }
      } ]
    },
    "favorited" : false,
    "retweeted" : false,
    "possibly_sensitive" : false,
    "filter_level" : "low",
    "lang" : "en"
  },
  "is_quote_status" : false,
  "quote_count" : 0,
  "reply_count" : 0,
  "retweet_count" : 0,
  "favorite_count" : 0,
  "entities" : {
    "hashtags" : [ {
      "text" : "SpiderMan",
      "indices" : [ 105, 115 ]
    } ],
    "urls" : [ ],
    "user_mentions" : [ {
      "screen_name" : "_Human_Thought_",
      "name" : "Gorzki_Aloes",
      "id" : 939609548536545280,
      "id_str" : "939609548536545280",
      "indices" : [ 3, 19 ]
    } ],
    "symbols" : [ ],
    "media" : [ {
      "id" : 1163952755037429767,
      "id_str" : "1163952755037429767",
      "indices" : [ 116, 139 ],
      "media_url" : "http://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
      "media_url_https" : "https://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
      "url" : "https://t.co/pCzeb7oEnV",
      "display_url" : "pic.twitter.com/pCzeb7oEnV",
      "expanded_url" : "https://twitter.com/_Human_Thought_/status/1163952756845142016/photo/1",
      "type" : "photo",
      "sizes" : {
        "thumb" : {
          "w" : 150,
          "h" : 150,
          "resize" : "crop"
        },
        "medium" : {
          "w" : 1024,
          "h" : 536,
          "resize" : "fit"
        },
        "small" : {
          "w" : 680,
          "h" : 356,
          "resize" : "fit"
        },
        "large" : {
          "w" : 1024,
          "h" : 536,
          "resize" : "fit"
        }
      },
      "source_status_id" : 1163952756845142016,
      "source_status_id_str" : "1163952756845142016",
      "source_user_id" : 939609548536545280,
      "source_user_id_str" : "939609548536545280"
    } ]
  },
  "extended_entities" : {
    "media" : [ {
      "id" : 1163952755037429767,
      "id_str" : "1163952755037429767",
      "indices" : [ 116, 139 ],
      "media_url" : "http://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
      "media_url_https" : "https://pbs.twimg.com/media/ECcw3SNX4AcJGgJ.jpg",
      "url" : "https://t.co/pCzeb7oEnV",
      "display_url" : "pic.twitter.com/pCzeb7oEnV",
      "expanded_url" : "https://twitter.com/_Human_Thought_/status/1163952756845142016/photo/1",
      "type" : "photo",
      "sizes" : {
        "thumb" : {
          "w" : 150,
          "h" : 150,
          "resize" : "crop"
        },
        "medium" : {
          "w" : 1024,
          "h" : 536,
          "resize" : "fit"
        },
        "small" : {
          "w" : 680,
          "h" : 356,
          "resize" : "fit"
        },
        "large" : {
          "w" : 1024,
          "h" : 536,
          "resize" : "fit"
        }
      },
      "source_status_id" : 1163952756845142016,
      "source_status_id_str" : "1163952756845142016",
      "source_user_id" : 939609548536545280,
      "source_user_id_str" : "939609548536545280"
    } ]
  },
  "favorited" : false,
  "retweeted" : false,
  "possibly_sensitive" : false,
  "filter_level" : "low",
  "lang" : "en",
  "timestamp_ms" : "1566546897934"
}*
### Sent tweets
*[ {
  "created_at" : "Fri Aug 23 08:39:11 +0000 2019",
  "id" : 1164819358809382912,
  "source" : "<a href=\"http://twitter.com/download/iphone\" rel=\"nofollow\">Twitter for iPhone</a>",
  "user" : "MapRecord[{utc_offset=null, friends_count=90, profile_image_url_https=https://pbs.twimg.com/profile_images/1049108781374660608/ztAnNPIo_normal.jpg, listed_count=0, profile_background_image_url=, default_profile_image=false, favourites_count=6843, description=100% California born to surf. My politics is Democrats. I believe in women having choice, and men staying out of it!, created_at=Mon Oct 08 01:08:50 +0000 2018, is_translator=false, profile_background_image_url_https=, protected=false, screen_name=TaylorPruitt222, id_str=1049104295029665792, profile_link_color=1DA1F2, translator_type=none, id=1049104295029665792, geo_enabled=false, profile_background_color=F5F8FA, lang=null, profile_sidebar_border_color=C0DEED, profile_text_color=333333, verified=false, profile_image_url=http://pbs.twimg.com/profile_images/1049108781374660608/ztAnNPIo_normal.jpg, time_zone=null, url=null, contributors_enabled=false, profile_background_tile=false, profile_banner_url=https://pbs.twimg.com/profile_banners/1049104295029665792/1538962001, statuses_count=6902, follow_request_sent=null, followers_count=103, profile_use_background_image=true, default_profile=true, following=null, name=Taylor Pruitt, location=California, USA, profile_sidebar_fill_color=DDEEF6, notifications=null}]",
  "geo" : null,
  "coordinates" : null,
  "place" : null,
  "quote_count" : "0",
  "reply_count" : "0",
  "retweet_count" : "0",
  "favourite_count" : null,
  "timestamp_ms" : "1566549551094"
} ]*












