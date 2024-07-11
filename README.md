# cf-custom-resource-populate-ddb
CloudFormation Template demonstrating usage of a Lambda-backed Custom Resource to pre-populate a DynamoDB table with hardcoded values

## Overview

### 1) DynamoDB Table Seed Data
- Table entires encoded as JSON and hardcoded into the CloudFormation Template via Mappings
- CF_MANAGED variable used to identify seed data entries
```
Mappings:
  MyTable:
    Table:
      Items: |
        [
          { "Id": "0", "Name":"Foo", "CF_MANAGED": true },
          { "Id": "1", "Name":"Bar", "CF_MANAGED": true }
        ]
```

### 2) Lambda-backed Custom Resource
- Executed initially when CloudFormation template is first deployed ('Create' Event)
- Assigning the 'Items' property to the value of the seed data ensures it is executed again whenever the data is modified ('Update' Event)
```
  PopulateMyTable:
    Type: Custom::PopulateDynamoDB
    Properties:
      ServiceToken: !GetAtt PopulateMyTableFunction.Arn
      Items: !FindInMap [ MyTable, Table, Items ]
      TableName: !Ref MyTable
      HashKey: 'Id'
```

### 3) Lambda Function
On Create/Update:
- Deletes seed data from DynamoDB table (CF_MANAGED=true)
- Adds seed data items to the DynamoDb table
```
    def resource_create(event, context):
        hash_key = event['ResourceProperties']['HashKey']
        items = json.loads(event['ResourceProperties']['Items'])
        dynamodb = boto3.resource('dynamodb')
        table_name = event['ResourceProperties']['TableName']
        table = dynamodb.Table(table_name)
        response = table.scan(FilterExpression=Attr('CF_MANAGED').eq(True))
        with table.batch_writer() as batch:
            for item in response['Items']:
                batch.delete_item( Key={ hash_key: item[hash_key] } )
        for item in items:
            table.put_item(Item=item)

    def resource_update(event, context):
        resource_create(event, context)

    def resource_delete(event, context):
        return
```