## Step 1:
-----------
Create a DynamoDB Table -- bankatmlocation -- PK -- locationId (s).
Enable Stream for this table.

## Lambda Code:
-----------------
`import json
import boto3
from algoliasearch.search_client import SearchClient
algolia_client = SearchClient.create("{App Name}", "{API Key}")
index = algolia_client.init_index("{Index/Algolia Table Name}")


client = boto3.client('location')

def lambda_handler(event, context):
    print(event)
    for i in event['Records']:
        if i['eventName']=='INSERT':
            locationId=i['dynamodb']['NewImage']['locationId']['S']
            name=i['dynamodb']['NewImage']['name']['S']
            line1=i['dynamodb']['NewImage']['line1']['S']
            line2=i['dynamodb']['NewImage']['line2']['S']
            city=i['dynamodb']['NewImage']['city']['S']
            state=i['dynamodb']['NewImage']['state']['S']
            country=i['dynamodb']['NewImage']['country']['S']
            zipCode=i['dynamodb']['NewImage']['zipCode']['S']
            complete_address=locationId+','+name+','+line1+','+line2+','+city+','+state+','+zipCode
            print("Complete Address : {}".format(complete_address))
            
            record = {"objectID": locationId, "name":name ,"line1":line1,"line2":line2,"city":city,"state":state,"country":country,"zipCode":zipCode}
            
            response = client.search_place_index_for_text(IndexName='{PLace Index name}', 
            FilterCountries=["IND"], 
            MaxResults=1, 
            Text=complete_address
            )
            
            location_response = response["Results"][0]['Place']['Geometry']['Point']
            record['_geoloc']={
                                "lat": location_response[1],
                                "lng": location_response[0]
                              }
            print(record)
            
            index.save_object(record).wait()
            
        elif i['eventName']=='REMOVE':
            location_id=i['dynamodb']['OldImage']['locationId']['S']
            print("Deleting the location ID : {}".format(location_id))
            index.delete_objects([location_id])`

## Step 2:
-----------
Add DynamoDB Access & AWS Location Service access to Lambda Role
Add DynamoDB Trigger


## Step 3:
-----------
Creating Lambda Layer:
-----------------------------
testalgolia02

sudo apt-get update
sudo apt install virtualenv
virtualenv -p /usr/bin/python3 algolia_test
source algolia_test/bin/activate
python3 --version  
sudo apt install python3-pip
python3 -m pip install --upgrade pip
mkdir -p lambda_layers/python/lib/python3.9/site-packages
cd lambda_layers/python/lib/python3.9/site-packages
python3 -m pip install --upgrade algoliasearch -t .
cd ~/lambda_layers
sudo apt install zip
zip -r algolia_layer1.zip *
sudo apt  install awscli
aws s3 cp  algolia_layer1.zip s3://testalgolia02/


## Step 4:
-----------
Run the DynamoDB Queries to ingest data in Algolia--

INSERT INTO "bankatmlocation" value {'locationId': 'Ground floor, 98',
'name': 'Krishnanagar Road',
'line1': 'Nabapally',
'line2': 'North 24 Parganas',
'city': 'Barasat',
'state': 'West Bengal',
'country': 'IND',
'zipCode': '700126'
}


INSERT INTO "bankatmlocation" value {
'locationId': 'No 16,GROUND FLOOR',
'name': 'Bhandari House',
'line1': 'NETAJI SUBHAS ROAD',
'line2': 'DALHOUSIE',
'city': 'KOLKATA',
'state': 'West Bengal',
'country': 'IND',
'zipCode': '700001'
}

INSERT INTO "bankatmlocation" value {
'locationId': '32',
'name': 'Jagat Banerjee Ghat Road',
'line1': 'Shibpur',
'line2': '',
'city': 'Howrah',
'state': 'West Bengal',
'country': 'IND',
'zipCode': '711102'
}

INSERT INTO "bankatmlocation" value {
'locationId': 'GROUND FLOOR, 535-B',
'name': 'GEETA GOPAL ROAD',
'line1': 'JAGADHARI ROAD',
'line2': 'AMBALA CANTT',
'city': 'AMBALA',
'state': 'HARYANA',
'country': 'IND',
'zipCode': '133001'
}



INSERT INTO "bankatmlocation" value
{
'locationId': '1st Floor',
'name': 'AG PLAZA',
'line1': 'gs road',
'line2': 'near Mizoram House',
'city': 'GUWAHATI',
'state': 'ASSAM',
'country': 'IND',
'zipCode': '781005'
}


Delete few items and observe the effect..

## Step 5:
----------
Create another Lambda to perform Location Based Search--

`import json

import boto3

from algoliasearch.search_client import SearchClient
algolia_client = SearchClient.create("EQRDTRCS6B", "bae4654ee4204e5415752773be7856c0")
index = algolia_client.init_index("dev_test")
client = boto3.client('location')

def lambda_handler(event, context):
    # TODO implement
    complete_address=json.loads(event['body'])['complete_address']
    print("Complete Address : {}".format(complete_address))
    
    response = client.search_place_index_for_text(IndexName='placatest', 
    FilterCountries=["IND"], 
    MaxResults=1, 
    Text=complete_address
    )
    
    longi,lat = response["Results"][0]['Place']['Geometry']['Point']
    

    results = index.search('', {
    'aroundLatLng': '{}, {}'.format(lat,longi),
    'aroundRadius' : 100000 })
    return results`

## Step 6:
---------
Create API using API Gateway

## Step 7:
----------
Perform end-to-end testing...

## Testcase 1:
--------------
{
"complete_address": "Kalka chownk, New Vita Enclave Rd, near vita milk plant, Ambala, Haryana 134003"
}

## Testcase 2:
--------------
{
"complete_address": "House no 2, Mother teresa road, Geetanagar, Auditek, bylane, Guwahati, Assam 781020"
}

## Testcase 3:
--------------
{
"complete_address": "1,2,3, Old Court House Street, Dalhousie Square, Kolkata, West Bengal 700069"
}

## Testcase 4:
--------------
{
"complete_address": "North, 400, Grand Trunk Rd, Salkia, Howrah, West
}