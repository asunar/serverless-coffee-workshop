+++
title = "Handling EventBridge Events with SQS"
weight = 14
+++

## Testing the "OrderCompleted" Rule

### Step-by-step instructions
8. Single Order Test
   - Create a new order. 
   - Check the `serverlesspresso-metrics-table`
   - ![Workflow Architecture](/images/se-mod4-items.png)
   - You’ll see 2 items: (Similar to screenshot above)
       - 2022-XX-XX#TotalSales
       - 2022-XX-XX#ItemName
   - These values are incremented each time there’s an `OrderManager.WaitingCompletion` event in the Event Bus.
9. Load Testing
  - We’ll now use CLI commands in Cloudshell to simulate a large number of orders and see how SQS can act as a buffer for event processing during traffic spikes.
  - These events will be matched with the OrderCompleted rule and sent to the MetricsQueue.
  - Before you run the load test, let's look at how Lambda will process messages off the queue.
  - Each invocation of the Lambda function will process the events in batches of up to 10 records or windows of 30 seconds. Reserved Concurrency defines how many concurrent Lambda executions would read messages off the queue, the default is up to 1000 concurrent executions at any given time. By using Reserved Concurrency, we’re saying there should only be 1 concurrent execution of this specific Lambda function. 
  - You can customize the ReservedConcurrentExecutions, BatchSize, and MaximumBatchingWindowInSeconds values to adjust how many records are processed at any given time, our current configuration means we'll process up to 10 records per Lambda execution.
  - You can check the serverless-order-metrics table items to see the metrics as they’re updated during the load test. You'll see four metrics; counts for each item type and a total order count. 
  - Use the following command in Cloudshell to simulate completed orders in `serverlesspresso-order-table`.

```
#!/bin/bash
drinks=("Latte" "Cappuccino" "Espresso")
for i in {1200..1500}

do
RANDOM=$$$(date +%s)
selecteddrink=${drinks[($RANDOM + $i) % 3]}
echo $selecteddrink
aws dynamodb put-item --table-name serverlesspresso-order-table --item "$(cat << EOF 
{
  "PK": {
      "S": "orders"
    },
  "SK": {
      "S": "$i"
    },
  "drinkOrder": {
    "S": "{\"userId\":\"$i\",\"drink\":\"$selecteddrink\"}"
  },
  "ORDERSTATE": {
    "S": "Completed"
  },
  "USERID": {
    "S": "$i"
  }
}
EOF
)" > /dev/null 2>&1

done
```

Use the following command in Cloudshell to simulate `OrderManager.OrderCompleted` events in the Event Bus.
```
#!/bin/bash
for i in {1200..1500}
do
aws events put-events --entries "$(cat << EOF 
{
    "Source":"awsserverlessda.serverlesspresso", 
    "EventBusName": "Serverlesspresso", 
    "DetailType":"OrderManager.OrderCompleted",
     "Detail": "{\"Message\": \"Barrista has cancelled or completed teh order\",\
                \"orderId\": \"$i\",\
                \"ORDERSTATE\": \"Completed\",\
                \"userId\": \"$i\"}"
}
EOF
)" > /dev/null 2>&1
echo "PutEvent for OrderId:" $i

done
```
- Open up the `PublishMetrics` Lambda function and navigate to the `Monitoring` tab. Expand the `PublishMetrics` function Invocations graph and select "Maximum" instead of "Sum". You'll see only 1 concurrent execution during the load test. Instead of scaling to process all the messages in the SQS Queue, Reserved Concurrency limited the function to only 1 concurrent execution.
- ![Invocations Graph](/images/se-mod4-invocations.png)
- Open up the`serverlesspresso-metrics-table` DynamoDB table and navigate to the `Monitor` tab. Expand the Write usage graph and select "Maximum" instead of "Sum". The Red line represents the provisioned capacity for the table whereas the blue line represents how many WCU's were consumed by the `PublishMetrics` function. Without the Reserved Concurrency configuration and SQS queue in place, the Lambda function would’ve scaled to many concurrent executions, consumed all of the Provisioned Write Capacity, and been throttled by the DynamoDB table as it tried to insert metrics.
- ![WCU Graph](/images/se-mod4-wcu.png)
## Takeaways
- We were able to provide business metrics for the investors by setting up a new rule and microservice without needing to modify the existing application stack. 
- Using SQS as a buffer between the Event Bus and Lambda prevents database throttling when updating downstream systems like DynamoDB.
- Using Cloudwatch metrics to assess the performance of your event driven design can help identify weak points in your architecture.