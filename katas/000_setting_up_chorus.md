# Kata 000: Setting up Chorus

We use a Docker-Compose based environment to manage and firing up all the components of the Chorus stack. Then you will have to load the data and configure the components.

Open up a terminal window and run:
> docker-compose up --build

It will take a while to download the data and build all the Docker images!  You may think it's frozen at various points, but be patient - it'll be up and running eventually.

We have XPack security activated for Elasticsearch. The default user `elastic`, with the role `superuser`, is provided as the default user. Use it for creating indexes, adding users, accessing Kibana for monitoring, etc.
We will create another user with the role `superuser`, `chorus_admin`, that we can use to access Elasticsearch and Kibana if we want to explore the data:

```
curl -u 'elastic:ElasticRocks' -X POST "localhost:9200/_security/user/chorus_admin?pretty" -H 'Content-Type: application/json' -d'
{
  "password" : "password",
  "roles" : ["superuser"]
}
'
```

Now we need to load our sample product data into Chorus.  Open a second terminal window, so you can see how the various system respond, as you work with Chorus.

Grab the sample dataset of ~19k products by running this command from the root directory of the Chorus checkout:

> wget https://querqy.org/datasets/icecat/icecat-products-w_price-19k-20201127.tar.gz

First, we have to uncompress the data:

> tar xzf icecat-products-w_price-19k-20201127.tar.gz

Next, we need to format the data before we can index it into Elasticsearch. For that run the script `transform_data.sh`:

> ./transform_data.sh > transformed_data.json

This can take a while. The script takes the freshly extracted JSON data and transforms it in a way, so it can be used by the [Elasticsearch's Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html).

Before we can index the data, we have to create an index with some predefined settings and a basic mapping:

> curl -u 'elastic:ElasticRocks' -s -X PUT "localhost:9200/ecommerce/" -H 'Content-Type: application/json' --data-binary @./elasticsearch/schema.json

With the index created and the data in a suitable format, we can go ahead and index the data into Elasticsearch:

> curl -u 'elastic:ElasticRocks' -X POST "localhost:9200/ecommerce/_bulk?pretty" -H 'Content-Type: application/json' --data-binary @transformed_data.json

The sample data will take only a couple of minutes to be indexed.

You can confirm that the data is indexed by visiting http://localhost:9200/ecommerce/_search. You should see some of the indexed documents with their product attributes.

For a more convenient way to see the indexed products, visit our mock e-commerce store, Chorus Electronics, available at http://localhost:4000/. Try out the facets, and search for something, like `coffee`.

Before we set up our search management UI (SMUI), we need to create documents, which contain an empty set of rules used by the standard Querqy query rewriters. They can later be utilized, by picking different relevance algorithms in our Online store.

```
curl -u 'elastic:ElasticRocks' -s --request PUT 'http://localhost:9200/_querqy/rewriter/common_rules' \
--header 'Content-Type: application/json' \
--data-raw '{
"class": "querqy.elasticsearch.rewriter.SimpleCommonRulesRewriterFactory",
"config": {
"rules": ""
}
}'
```
```
curl -u 'elastic:ElasticRocks' -s --request PUT 'http://localhost:9200/_querqy/rewriter/common_rules_prelive' \
--header 'Content-Type: application/json' \
--data-raw '{
"class": "querqy.elasticsearch.rewriter.SimpleCommonRulesRewriterFactory",
"config": {
"rules": ""
}
}'
```
```
curl -u 'elastic:ElasticRocks' -s --request PUT 'http://localhost:9200/_querqy/rewriter/replace' \
--header 'Content-Type: application/json' \
--data-raw '{
"class": "querqy.elasticsearch.rewriter.ReplaceRewriterFactory",
"config": {
"rules": ""
}
}'
```
```
curl -u 'elastic:ElasticRocks' -s --request PUT 'http://localhost:9200/_querqy/rewriter/replace_prelive' \
--header 'Content-Type: application/json' \
--data-raw '{
"class": "querqy.elasticsearch.rewriter.ReplaceRewriterFactory",
"config": {
"rules": ""
}
}'
```

You can now choose between three different algorithms in the frontend: `Default`, `Querqy Prelive`, `Querqy Live`. They will become relevant (pun intended) in later Katas.  

We also need to set up the name of the index we're going to do active search management for in SMUI. We do this via:

```sh
curl -X PUT -H "Content-Type: application/json" -d '{"name":"ecommerce", "description":"Ecommerce Demo"}' http://localhost:9000/api/v1/solr-index
```

Grab the `returnId` from the response, something like `3f47cc75-a99f-4653-acd4-a9dc73adfcd1`, you'll need it for the next steps!

```sh
export ES_INDEX_ID=5bc6e917-33b7-45ec-91ba-8e2c4a2e8085
curl -X PUT -H "Content-Type: application/json" -d '{"name":"product_type"}' http://localhost:9000/api/v1/${ES_INDEX_ID}/suggested-solr-field
curl -X PUT -H "Content-Type: application/json" -d '{"name":"title"}' http://localhost:9000/api/v1/${ES_INDEX_ID}/suggested-solr-field
```

Now please check that SMUI is up and running, by visiting http://localhost:9000.

Don't get confused by any references to Solr in the UI. SMUI was designed as a search management UI for Solr. And the usage for OpenSearch is not fully supported yet. Contributions are very welcome!

We'll learn more about how to use SMUI later, however test that it's working by clicking the `Publish to LIVE` button and confirming the publication by hitting `Yes, publish to LIVE`. You will get a confirmation message, that the rules were deployed successfully.

Now we want to pivot to setting up our Offline Testing Environment. Today we have Quepid integrated into this edition of Chorus.

First we need to create the database for Quepid:

> docker-compose run --rm quepid bin/rake db:setup

We also need to create an account with Administrator permissions:

> docker-compose run quepid thor user:create -a admin@choruselectronics.com "Chorus Admin" password

Visit Quepid at http://localhost:3000 and log in with the email and password you just set up.

Go through the initial case setup process. Quepid will walk you through setting up a _Movie Cases_ case via a Wizard interface, and then shows you some of the key features of Quepid's UI.  I know you want to skip the tour of the Quepid interface, however there is a lot of interactivity in the UI, so it's recommended to go through the tutorial to get acquainted! As this is the Chorus for OpenSearch project, you can choose OpenSearch as your search engine instead of Solr.

Congratulations! You now have Chorus - The OpenSearch Edition successfully running with its components!
