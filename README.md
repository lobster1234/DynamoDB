# Setup and QuickStart

This is a quick and dirty primer on how to get set up with DynamoDB on your (Mac or Linux) laptop.

### Downloading and bringing up the server

* First off, download the server from there - http://dynamodb-local.s3-website-us-west-2.amazonaws.com/dynamodb_local_latest.tar.gz

* Once downloaded, do a ```tar -xvfz dynamodb_local_latest.tar.gz``` in the folder you downloaded it to.

* ```cd dynamodb_local_*```

* ```java -jar java -Djava.library.path=./DynamoDBLocal -jar DynamoDBLocal.jar -dbPath /tmp```

* **If you're behind a proxy, PLEASE set no proxy for localhost or you'll waste an hour like I did with a cryptic error from AWS CLI. To do this, ```export no_proxy=localhost```**

* Now fire up your AWS CLI. Type this on the command prompt and you'll know if you have it installed. Download it from here if needed - http://docs.aws.amazon.com/cli/latest/userguide/installing.html
```
aws
usage: aws [options] <command> <subcommand> [parameters]
aws: error: too few arguments
```

### Creating a table

* Now we'd want to see if there is anything in there already. We should get an empty list.
```
aws dynamodb list-tables  --endpoint-url http://localhost:8000 
{
    "TableNames": []
}
```

* Lets create a table called Persons, and we will add id (numeric) as a primary key, and name as the string field. We will sort by name. You will find this to be the most ass backwards syntax compared to mongo or PL-SQL, but bear with me and it will all make sense.

```
aws dynamodb create-table --endpoint-url http://localhost:8000 
--table-name Persons 
--attribute-definitions AttributeName=id,AttributeType=N AttributeName=name,AttributeType=S 
--key-schema AttributeName=id,KeyType=HASH AttributeName=name,KeyType=RANGE 
--provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
```
If you've copy pasted the line above correctly, you'll see the below output.
```json
{
    "TableDescription": {
        "AttributeDefinitions": [
            {
                "AttributeName": "id", 
                "AttributeType": "N"
            }, 
            {
                "AttributeName": "name", 
                "AttributeType": "S"
            }
        ], 
        "ProvisionedThroughput": {
            "NumberOfDecreasesToday": 0, 
            "WriteCapacityUnits": 1, 
            "LastIncreaseDateTime": 0.0, 
            "ReadCapacityUnits": 1, 
            "LastDecreaseDateTime": 0.0
        }, 
        "TableSizeBytes": 0, 
        "TableName": "Persons", 
        "TableStatus": "ACTIVE", 
        "KeySchema": [
            {
                "KeyType": "HASH", 
                "AttributeName": "id"
            }, 
            {
                "KeyType": "RANGE", 
                "AttributeName": "name"
            }
        ], 
        "ItemCount": 0, 
        "CreationDateTime": 1462520065.493
    }
}
```

If you did not set the no_proxy, you will see this error -
```
No JSON object could be decoded
```

* Now lets see if this table shows up when we run list-tables with ```aws dynamodb list-tables  --endpoint-url http://localhost:8000 ```
```json
{
    "TableNames": [
        "Persons"
    ]
}
```

* If you want details on the table structure, you can use the following command, which will return the table metadata in JSON format.
```
aws dynamodb describe-table --table-name Persons  --endpoint-url http://localhost:8000 
```

### Adding data to our table

* So far so good, now lets go ahead add some rows to this table. Amazon calls them items. **DynamoDB does an upsert, which means if the primary key you specified already exists, the item will be replaced with what you provide.**

* Lets add a person Manish with id as 1.

```
aws dynamodb put-item --table-name Persons  
--item '{"id":{"N":"1"},"name":{"S":"Manish"}}' 
--endpoint-url http://localhost:8000 
```

Lets add few more items

```
aws dynamodb put-item --table-name Persons  
--item '{"id":{"N":"2"},"name":{"S":"Tim"}}' 
--endpoint-url http://localhost:8000 

aws dynamodb put-item --table-name Persons  
--item '{"id":{"N":"3"},"name":{"S":"Jack"}}' 
--endpoint-url http://localhost:8000 
```

### Scan vs. Query
DynamoDB supports Scanning as well as Querying. A Scan reads all items (returns max of 1 MB data), while a query goes by only primary key attribute values. This distinction is very important. Read [this](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/QueryAndScan.html) for details.

### Scanning the table

* We get all items here using a scan table
```
aws dynamodb scan --table-name=Persons --endpoint-url http://localhost:8000
```
The output will be as below - 
```json
{
    "Count": 3, 
    "Items": [
        {
            "name": {
                "S": "Tim"
            }, 
            "id": {
                "N": "2"
            }
        }, 
        {
            "name": {
                "S": "Manish"
            }, 
            "id": {
                "N": "1"
            }
        }, 
        {
            "name": {
                "S": "Jack"
            }, 
            "id": {
                "N": "3"
            }
        }
    ], 
    "ScannedCount": 3, 
    "ConsumedCapacity": null
}
```

### Pagination and Limits

We can control the number of items returned using ```--max-items``` argument as below -
```
aws dynamodb scan --table-name=Persons 
--max-items 2  
--endpoint-url http://localhost:8000
```
This will only return 2 items instead of all 3 we have in the table.
```json
{
    "Count": 3, 
    "Items": [
        {
            "name": {
                "S": "Tim"
            }, 
            "id": {
                "N": "2"
            }
        }, 
        {
            "name": {
                "S": "Manish"
            }, 
            "id": {
                "N": "1"
            }
        }
    ], 
    "NextToken": "None___2", 
    "ScannedCount": 3, 
    "ConsumedCapacity": null
}
```
Note the _NextToken_ above, which is the key to pagination. 


### Getting only certain attributes, and handling reserved keywords

In order to get only a list of attributes, we use projections. Interestingly, we have used _name_ which is a reserved keyword, so we have to "substitute" it. Enjoy the syntax of the below where we project to get only name, and use expression attribute names to handle our _name_ situation.

```
aws dynamodb scan --table-name=Persons 
--projection-expression "#nm" 
--expression-attribute-names '{"#nm" : "name" }' 
--endpoint-url http://localhost:8000
```

The output will only have _name_ attribute and not _id_ since we only projected _name_.

```json
{
    "Count": 3, 
    "Items": [
        {
            "name": {
                "S": "Tim"
            }
        }, 
        {
            "name": {
                "S": "Manish"
            }
        }, 
        {
            "name": {
                "S": "Jack"
            }
        }
    ], 
    "ScannedCount": 3, 
    "ConsumedCapacity": null
}
```

### Getting a count of items
We use ```--select COUNT``` to get the number of items matched. 
```
aws dynamodb scan --table-name=Persons 
--select COUNT --endpoint-url http://localhost:8000
```
The output will contain the _Count_ field with the number of items.
```json
{
    "Count": 3, 
    "Items": [], 
    "ScannedCount": 3, 
    "ConsumedCapacity": null
}
```


### Getting an item by primary key

### Using query filters


### Updating an item

### Deleting an item

### Dropping the table

# Further Reading

Now that you've come this far, here are some interesting topics for you to explore -
* Using pagination for scanning and querying
* Batch inserts and updates
* Provisioned Read/Writes in AWS (not your laptop as these fields are ignored)

# References

* AWS Documentation - http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html
* CLI for DynamoDB - http://docs.aws.amazon.com/cli/latest/reference/dynamodb/index.html
