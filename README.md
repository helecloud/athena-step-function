# athena-step-function-query

This is a AWS Servless Application, that consists of a Step Functions State Machine that runs Athena queries without any idling lambdas with configurable retries for error handling. Idea is based on github.com/burtcorp/athena-runner

In case of error, the query is ran again via [Decorrelated Jitter Algorithm](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

## Why?

You probably need to consider using this if the answer to any of the following questions is true:

* Do you want to ensure that your Athena query is retried in case of failure(not everything is always working)?
* Do you want to run Athena queries based of events?
* Do you want to run Athena queries in your AWS Lambda based application but can't have your Lambda function waiting for the query since your query takes more than 15 minutes?

## Step Function Input

The step function takes the following parameters as input:

* StartQueryParameters - These are the parameters for [StartQueryExecution API](https://docs.aws.amazon.com/athena/latest/APIReference/API_StartQueryExecution.html).
* InitialWaitTime - Optional - Time in seconds for the first period of waiting after starting the function. If you know your query takes a certain amount of time at minimum set this to that.
* BaseWaitTime - Optional - Time in seconds for waiting in between checking query results. Increasing this would reduce the cost of running the Lambdas to check the query results.
* WaitMultiplier - Optional - Set this to higher than 1 in case if you want increasing wait times, between checking results.
* MaxWait - Optional - This will be the maximum amount of time in seconds the functions wait in between query results. This should be lesser than 30 minutes to avoid cold starts.
* MaxRetries - Optional - Maximum number of retries in case of failure.
* RetryStateChangeReasonMatch - Optional - If you specify this, then the query only will be attempted if the StateChangeReason from the API result matches the provided Python RegExp

The above parameters defaults are set by the Cloudformation template.

## Step Function Output

The step function will have an output in the following format:

* StartQueryExecutionResult: This is the result of [athena:StartQueryExecution API call](https://docs.aws.amazon.com/athena/latest/APIReference/API_StartQueryExecution.html)
* Result: This is the result of [athena:GetQueryExecution API call](https://docs.aws.amazon.com/athena/latest/APIReference/API_GetQueryExecution.html) 

It will also contain the provided input parameters and a few other properties that are used by the state machine.
 

## Example Run

#### Input

```json
{
  "StartQueryParameters": {
    "QueryString": "MSCK REPAIR TABLE example_table;",
    "ResultConfiguration": {
      "OutputLocation": "s3://some-bucket/optional-path"
    }
  }
}
```

#### Output

```json
{
  "StartQueryParameters": {
    "QueryString": "MSCK REPAIR TABLE example_table;",
    "ResultConfiguration": {
      "OutputLocation": "s3://some-bucket/optional-path"
    }
  },
  "StartQueryExecutionResult": {
    "QueryExecutionId": "46140486-fa0a-4017-bd87-2f6b1eec0b30",
    "ResponseMetadata": {
      "RequestId": "3d0b86fe-9a87-4568-b9ce-b1a9412cc1c5",
      "HTTPStatusCode": 200,
      "HTTPHeaders": {
        "content-type": "application/x-amz-json-1.1",
        "date": "Fri, 14 Aug 2020 15:25:50 GMT",
        "x-amzn-requestid": "3d0b86fe-9a87-4568-b9ce-b1a9412cc1c5",
        "content-length": "59",
        "connection": "keep-alive"
      },
      "RetryAttempts": 0
    }
  },
  "WaitTime": "15",
  "Result": {
    "QueryExecutionId": "46140486-fa0a-4017-bd87-2f6b1eec0b30",
    "Query": "MSCK REPAIR TABLE example_table;",
    "StatementType": "DDL",
    "ResultConfiguration": {
      "OutputLocation": "s3://some-bucket/optional-path/46140486-fa0a-4017-bd87-2f6b1eec0b30.txt"
    },
    "QueryExecutionContext": {},
    "Status": {
      "State": "SUCCEEDED",
      "SubmissionDateTime": "2020-08-14 15:25:51.061000+00:00",
      "CompletionDateTime": "2020-08-14 15:25:53.088000+00:00"
    },
    "Statistics": {
      "EngineExecutionTimeInMillis": 1840,
      "DataScannedInBytes": 0,
      "TotalExecutionTimeInMillis": 2027,
      "QueryQueueTimeInMillis": 168,
      "ServiceProcessingTimeInMillis": 19
    },
    "WorkGroup": "primary"
  },
  "Decision": "SUCCEED"
}
```
