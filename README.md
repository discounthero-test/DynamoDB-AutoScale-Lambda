# Dynamo-AutoScale-Lambda
Automatically scales DynamoDB capacity up and down

## Coming shortly

Shortly I will update this repo with some updates I've made. These changes are currently being tested.

The first change is to split the IncrementBuffer into read and write values to give more control. 

Secondly I've added a config field called NoScaleDownHours, which is an array of numbers. Adding values here prevents any scaling down during those hours (UTC). This is useful during multiple hour spikes where you want to delay the scale down until traffic has properly settled at normal levels. 

Finally I've added an IncrementBuffer modifier allowing you to modify the increment buffer at certain hours of the day. For example I can say I want the buffer to increase by 30 value during certain hours of the day. You can do this for reads and/or writes per table. The value here is if you have predictable traffic spikes, you can artifically scale up at certain times, essentially being proactive rather than reactive. This adds value as it can sometimes take a little time to scale up to the required capacity, especially if you have complex scenarios where multiple tables can be bottlenecks for each other.

## Purpose

A simple solution to effectively scale DynamoDB capacity provisioning. Other solutions exist, however I did not find them suitable for my requirements. The project that forced this requirement was clickstream processing, requiring highly responsive scaling, especially when scaling up.

![Scaling](scale1.PNG)

![Scaling](scale2.PNG)

## Usage

1. Get the code.
2. Run npm install.
3. Review config.js. See Config section below for info.
4. Zip all of the files.
5. Create an IAM role. See Role section below for info.
5. Create a new AWS Lambda function.
	1. Skip blueprint selection.
	2. Enter a function name.
	3. Ensure Node.js 4.3 is selected as the Runtime
	4. Upload the zip you created earlier.
	5. Leave the Handler as index.handler.
	6. Choose the role you created earlier.
	7. For Memory, chose the max value.
	8. For timeout, typical duration for processing 20 tables is around 1 second. Set a timeout appropriate to your usage.
	9. Add an event source of type CloudWatch Events - Schedule. Set a rate of 1 minute.
	
## Role

Create a new IAM role. Attach the managed policy named 'AWSLambdaBasicExecutionRole'. Then create a role policy with the following statement:

~~~~
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "dynamodb:ListTables",
                "dynamodb:DescribeTable",
                "dynamodb:UpdateTable",
                "dynamodb:Scan",
                "cloudwatch:GetMetricStatistics",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
~~~~

## Config

The function is driven by whatever settings you place in config.js (or DynamoDB, see below). The code comes as default with a single config line that enables scaling for all Dynamo Tables in the deployed region:

`{ Search: '.*', UseRegex: true, MinReads: 1, MaxReads: 50, MinWrites: 1, MaxWrites: 50, AssessmentMinutes: 2, IncrementBuffer: 5, DecrementPercentBarrier: 65, DecrementMinutesBarrier: 60 }`

There is a commented out example of a specific table config row also included. The above catch-all config line should always be placed last in the list.

**Search** - The value to search for. Either a table name or regex to match a table name.

**UseRegex** - A boolean indicating if regex should be used to match the Search field with table names.

**MinReads** - The minimum number of reads a table or index should have.

**MaxReads** - The maximum number of writes a table or index should have.

**MinWrites** - The minimum number of reads a table or index should have.

**MaxWrites** - The maximum number of writes a table or index should have.

**AssessmentMinutes** - The number of minutes of CloudWatch metrics to average in order to determine the current consumption. Do not set below 2. Querying CloudWatch for 1 minutes of metrics often results in no data, which may result in scaling down. This value strongly affects the sensitivity of scaling. A higher value may result in short term spikes being ignored.

**IncrementBuffer** - The amount to scale up by. Also used as a proximity measure, meaning if the consumed value plus the buffer is above the capacity, it will take that to mean it should scale up.

**DecrementPercentBarrier** - Prevents scaling down of capacity until the consumed capacity is below this percentage of the current capacity. Geared towards protecting unneccessary scale downs. Ignored when dealing within 10 of the MinReads value.

**DecrementMinutesBarrier** - The minimum number of minutes that must have gone by since the last capacity decrement before another can occur.


In addition to these settings, the code stops scaling down based on the time of day. This is because you can only scale a table down 4 times within a UTC day. The function seeks to spread capacity decreases intelligently throughout the day so you don't end up with 20 hours stuck an unneccessarily high capacity.

## Store Config in Dynamo

If you need to change your config.js variables frequently, it may be desirable to put the config into DynamoDB rather that amending the deployed code. Note that if you decide to use DynamoDB, you probably want to empty out the items array in config.js. The function will load config.js first and then append config dynamo items after it. Order matters here, so config.js rules will be processed first. By default config.js contains a catch-all rule, so unless you remove it, your rules in dynamo won't have any effect.

To use dynamo for config, just set the externalConfigTableName variable at the top of the code to the name of your dynamo table containing config info. The table should have a partion key (hash) of type string with a name of "Search". It should have a sort key (range) of type number with a name of Order. An example item is detailed below.

![Dynamo](DynamoConfig.PNG)

The order value is used to determine the order by which items are loaded into the config.items array.

## Disclaimer

In no event will I be liable for any loss or damage including without limitation, indirect or consequential loss or damage, or any loss or damage whatsoever arising from loss of data or profits arising out of, or in connection with, the use of this code.

