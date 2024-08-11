---
title: "Migrating Data Between DynamoDB Tables Across Different AWS Accounts"
date: "2024-08-10T00:00:00.000Z"
template: "post"
draft: false
slug: "/posts/migrating-data-between-dynamodb-tables"
category: "Tech Guides"
tags:
  - "AWS"
  - "DynamoDB"
  - "Data Migration"
description: "Learn how to migrate data between DynamoDB tables across different AWS accounts using a Python script with the boto3 library."
socialImage: "./media/dynamodb-migration.jpg"
---

When working with AWS's DynamoDB, there may come a time when you need to migrate data from one table to another across different AWS accounts. In this post, we will explore a Python script that leverages the boto3 library to handle this task smoothly and efficiently.

To start, let's take a look at the code:

```python
import boto3
import time

# Configure source and destination AWS account profiles
source_profile = 'default'
destination_profile = 'target-profile'

# Set source and destination table names
source_table_name = 'table-1'
destination_table_name = 'table-2'

# Initialize DynamoDB client for both accounts
source_session = boto3.Session(profile_name=source_profile)
destination_session = boto3.Session(profile_name=destination_profile)

source_dynamodb = source_session.resource('dynamodb')
destination_dynamodb = destination_session.resource('dynamodb')

# Check if the destination table exists
try:
    destination_dynamodb.Table(destination_table_name).load()
except destination_dynamodb.exceptions.ResourceNotFoundException:
    print(
        f"Table {destination_table_name} not found in the destination account.")
    exit(1)

# Initialize source and destination tables
source_table = source_dynamodb.Table(source_table_name)
destination_table = destination_dynamodb.Table(destination_table_name)

# Initialize counters
source_item_count = 0
destination_item_count = 0

# Scan the source table and insert items into the destination table
scan = source_table.scan()
source_item_count += scan['Count']

with destination_table.batch_writer(overwrite_by_pkeys=['id']) as batch:
    for item in scan['Items']:
        try:
            batch.put_item(Item=item)
            destination_item_count += 1
        except destination_dynamodb.meta.client.exceptions.ProvisionedThroughputExceededException:
            print(
                'ProvisionedThroughputExceededException occurred, sleeping for 1 second.')
            time.sleep(1)

    # If the scan returned 1MB of data, continue scanning from where we left off
    while 'LastEvaluatedKey' in scan:
        scan = source_table.scan(ExclusiveStartKey=scan['LastEvaluatedKey'])
        source_item_count += scan['Count']

        for item in scan['Items']:
            try:
                batch.put_item(Item=item)
                destination_item_count += 1
            except destination_dynamodb.meta.client.exceptions.ProvisionedThroughputExceededException:
                print(
                    'ProvisionedThroughputExceededException occurred, sleeping for 1 second.')
                time.sleep(1)

print(
    f"Data migration from {source_table_name} to {destination_table_name} is complete.")
print(f"Total items in {source_table_name} is {source_item_count}")
print(f"Total items in {destination_table_name} is {destination_item_count}")
```

## Code Explanation

This script essentially copies data from a source DynamoDB table to a destination DynamoDB table. The tables could belong to different AWS accounts, hence the need for specifying source and destination profiles.

### Initialization

The script starts by importing the necessary libraries, `boto3` for interacting with AWS resources and `time` for handling rate limiting later on. It then sets up the necessary configurations: AWS account profiles (`source_profile`, `destination_profile`) and table names (`source_table_name`, `destination_table_name`).

### Establishing Sessions and Clients

With the configurations set, the script initializes the `boto3` sessions for each account and gets the DynamoDB resources.

### Validating the Destination Table

Before starting the migration, it checks if the destination table exists by trying to load the table. If the table does not exist, it raises a `ResourceNotFoundException`, and the script ends with an error message.

### Table Scanning and Data Migration

With the validation done, the script then initializes the DynamoDB tables using the established sessions. A scan operation is performed on the source table, and the count of items is incremented in the `source_item_count` counter.

The scan results are then written to the destination table using a batch writer. If the script encounters a `ProvisionedThroughputExceededException`, which may occur if you exceed your table's provisioned throughput, it will pause for a second before continuing.

The script also checks if there are more items to be scanned in the source table (DynamoDB returns up to 1 MB of data for each Scan operation). If more items exist, it continues the scan operation from where it left off.

### Migration Summary

Finally, the script prints a summary of the data migration process, including the total items in the source and destination tables, confirming the completion of the data migration.

## Considerations

While this script is useful for smaller migrations, be aware that large-scale migrations might require a different approach, such as using AWS Data Pipeline or DMS (Database Migration Service), which are designed for handling large data transfers efficiently and reliably.

Also, remember that this script assumes that both tables have the same schema. If your tables have different schemas, you might need to adapt the script to map the source data to the destination data appropriately.

And there you have it! You now have a tool to migrate data between DynamoDB tables across different AWS accounts.
