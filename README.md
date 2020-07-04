# invokust

A tool for running [Locust](http://locust.io/) load tests from within Python without the need to use the locust command line. This gives more flexibility for automation such as QA/CI/CD tests and also makes it possible to run locust on [AWS Lambda](https://aws.amazon.com/lambda/) for ultimate scalability.

## Installation

Install via pip:

```
pip3 install invokust
```

## Examples

Running a load test using a locust file:

```python
import invokust

settings = invokust.create_settings(
    locustfile='locustfile_example.py',
    host='http://example.com',
    num_clients=1,
    hatch_rate=1,
    run_time='3m'
    )

loadtest = invokust.LocustLoadTest(settings)
loadtest.run()
loadtest.stats()
'{"num_requests_fail": 0, "num_requests": 10, "success": {"/": {"median_response_time": 210, "total_rpm": 48.13724156297892, "request_type": "GET", "min_response_time": 210, "response_times": {"720": 1, "210": 7, "820": 1, "350": 1}, "num_requests": 10, "response_time_percentiles": {"65": 210, "95": 820, "75": 350, "85": 720, "55": 210}, "total_rps": 0.802287359382982, "max_response_time": 824, "avg_response_time": 337.2}}, "locust_host": "http://example.com", "fail": {}, "num_requests_success": 10}'
```

Running a load test without locust file:

```python
import invokust

from locust import HttpUser, between, task

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)

    @task()
    def get_home_page(self):
        '''
        Gets /
        '''
        self.client.get("/")

settings = invokust.create_settings(
    classes=[WebsiteUser],
    host='http://example.com',
    num_users=1,
    hatch_rate=1,
    run_time='3m'
)

loadtest = invokust.LocustLoadTest(settings)
loadtest.run()
loadtest.stats()
'{"num_requests_fail": 0, "num_requests": 10, "success": {"/": {"median_response_time": 330, "total_rpm": 40.53552561950598, "request_type": "GET", "min_response_time": 208, "response_times": {"230": 1, "1000": 1, "780": 1, "330": 1, "1100": 1, "210": 3, "790": 1, "670": 1}, "num_requests": 10, "response_time_percentiles": {"65": 780, "95": 1100, "75": 790, "85": 1000, "55": 670}, "total_rps": 0.6755920936584331, "max_response_time": 1111, "avg_response_time": 552.8}}, "locust_host": "http://example.com", "fail": {}, "num_requests_success": 10}'
```

## Running Locust on AWS Lambda

<img src="http://d0.awsstatic.com/Graphics/lambda-icon-smallr1.png" alt="Lambda logo" height="100"><img src="http://locust.io/static/img/logo.png" alt="Locust logo" height="100">

[AWS Lambda](https://aws.amazon.com/lambda/) is a great tool for load testing as it is very cheap (or free) and highly scalable.

There are many load testing tools such as [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) and [wrk](https://github.com/wg/wrk). Then there are other cloud based load testing options such as [BlazeMeter](https://www.blazemeter.com/) or [Loader](https://loader.io/) and some more DIY solutions that use AWS Lambda too such as [Goad](https://goad.io/) or [serverless-artillery](https://github.com/Nordstrom/serverless-artillery). But these all have the same drawback: They are too simplistic. They can perform simple GET or POST requests but can't accurately emulate more complex behaviour. e.g. browsing a website, selecting random items, filling a shopping cart and checking out. But with [Locust](http://locust.io/) this is possible.

Included is an example function for running Locust on AWS Lambda, `lambda_locust.py`.

### Creating a Lambda function

The process for running a locust test on Lambda involves [creating a zip file](http://docs.aws.amazon.com/lambda/latest/dg/lambda-python-how-to-create-deployment-package.html) of the locust load test, creating a Lambda function and then triggering the function.

Install invokust (and its dependencies) python packages locally:

```
pip3 install invokust --target=python-packages
```

Or if running on a Mac (python packages need to be compiled for 64 bit Linux) you can use docker:

```
docker run -it --volume=$PWD/python-packages:/python-packages python:3.6 bash -c "pip install invokust --target=/python-packages"
```

Create the zip file:

```
zip -q -r lambda_locust.zip lambda_locust.py locustfile_example.py python-packages
```

Then create the Lambda function using using the AWS CLI:

```
aws lambda create-function --function-name lambda_locust --timeout 300 --runtime python3.6 --role arn:aws:iam::9999999999:role/lambda_basic_execution --handler lambda_locust.handler --zip-file fileb://lambda_locust.zip
```

Or [Terraform](https://www.terraform.io/) and the example [main.tf](main.tf) file:

```
terraform apply
...
```

### Invoking the function

The Locust settings can be passed to the Lambda function or can be set from environment variables. The environment variables are:

  - LOCUST_LOCUSTFILE: Locust file to use for the load test
  - LOCUST_CLASSES: Names of locust classes to use for the load test (instead of a locustfile). If more than one, separate with comma.
  - LOCUST_HOST: The host to run the load test against
  - LOCUST_NUM_CLIENTS: Number of clients to simulate
  - LOCUST_HATCH_RATE: Number of clients per second to start
  - LOCUST_RUN_TIME: The time the test should run for
  - LOCUST_LOGLEVEL: Level of logging

[AWS CLI](https://aws.amazon.com/cli/) example with Locust settings in a payload:

```
aws lambda invoke --function-name lambda_locust --invocation-type RequestResponse --payload '{"locustfile": "locustfile_example.py", "host":"https://example.com", "num_requests":"20", "num_clients": "1", "hatch_rate": "1", "run_time":"3m"}' output.txt
{
    "StatusCode": 200
}
cat output.txt
"{\"success\": {\"GET_/\": {\"request_type\": \"GET\", \"num_requests\": 20, \"min_response_time\": 86, \"median_response_time\": 93 ...
```

Python boto3 example:

```python
import json
from boto3.session import Session
from botocore.client import Config

session = Session()
config = Config(connect_timeout=10, read_timeout=310)
client = session.client('lambda', config=config)

lambda_payload = {
    'locustfile': 'locustfile_example.py',
    'host': 'https://example.com',
    'num_clients': '1',
    'hatch_rate': 1,
    'run_time':'3m'
}

response = client.invoke(FunctionName='lambda_locust', Payload=json.dumps(lambda_payload))
json.loads(response['Payload'].read())
'{"success": {"GET_/": {"request_type": "GET", "num_requests": 20, "min_response_time": 87, "median_response_time": 99, "avg_response_time": 97.35 ...
```

### Running a real load test

Lambda function execution time is limited to a maximum of 5 minutes. To run a real load test the function will need to be invoked repeatedly and likely in parallel to generate enough load. To manage this there is a class called `LambdaLoadTest` that can manage invoking the function in parallel loops and collecting the statistics.

```python
import logging
from invokust.aws_lambda import LambdaLoadTest

logging.basicConfig(level=logging.INFO)

lambda_payload = {
    'locustfile': 'locustfile_example.py',
    'host': 'https://example.com',
    'num_clients': '1',
    'hatch_rate': 1,
    'run_time':'3m'
}

load_test = LambdaLoadTest(
  lambda_function_name='lambda_locust',
  threads=2,
  ramp_time=0,
  time_limit=30,
  lambda_payload=lambda_payload
)

load_test.run()
print(load_test.get_summary_stats())
```

The output:
```
INFO:root:
Starting load test...
Function name: lambda_locust
Ramp time: 0s
Threads: 2
Lambda payload: {'locustfile': 'locustfile_example.py', 'host': 'https://example.com', 'num_clients': '1', 'hatch_rate': 1, 'run_time': '3m'}
Start ramping down after: 30s
INFO:root:thread started
INFO:root:Invoking lambda...
INFO:root:threads: 1, rpm: 0, time elapsed: 0s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 1, rpm: 0, time elapsed: 3s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:thread started
INFO:root:Invoking lambda...
INFO:root:threads: 2, rpm: 0, time elapsed: 6s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 9s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 12s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 15s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 18s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 21s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 24s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 27s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 30s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:threads: 2, rpm: 0, time elapsed: 33s, total requests from finished threads: 0, request fail ratio: 0, invocation error ratio: 0
INFO:root:Time limit reached. Starting ramp down...
INFO:root:Waiting for all Lambdas to return. This may take up to 3m.
INFO:invokust.aws_lambda.lambda_load_test:Lambda invocation complete. Requests (errors): 1867 (0), execution time: 180066ms, sleeping: 0s
INFO:root:thread finished
INFO:invokust.aws_lambda.lambda_load_test:Lambda invocation complete. Requests (errors): 1884 (0), execution time: 180065ms, sleeping: 0s
INFO:root:thread finished
{'lambda_invocation_count': 2, 'total_lambda_execution_time': 360131, 'requests_total': 3751, 'request_fail_ratio': 0.0, 'invocation_error_ratio': 0.0}
```

There is also an example CLI tool for running a load test, `invokr.py`:

```
$ ./invokr.py --function_name=lambda_locust --locust_file=locustfile_example.py --locust_host=https://example.com --threads=1 --time_limit=15 --locust_users=2
2017-05-22 20:16:22,432 INFO   MainThread
Starting load test
Function: lambda_locust
Ramp time: 0
Threads: 1
Lambda payload: {'locustfile': 'locustfile_example.py', 'host': 'https://example.com', 'num_users': 2, 'hatch_rate': 10, 'run_time': '15s'}

[2020-06-28 19:58:22,103] pudli/INFO/root: thread started
[2020-06-28 19:58:22,107] pudli/INFO/root: threads: 1, rpm: 0, run_time: 0, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:25,108] pudli/INFO/root: threads: 1, rpm: 0, run_time: 3, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:28,109] pudli/INFO/root: threads: 1, rpm: 0, run_time: 6, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:31,110] pudli/INFO/root: threads: 1, rpm: 0, run_time: 9, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:34,112] pudli/INFO/root: threads: 1, rpm: 0, run_time: 12, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:37,113] pudli/INFO/root: threads: 1, rpm: 0, run_time: 15, requests_total: 0, request_fail_ratio: 0, invocation_error_ratio: 0
[2020-06-28 19:58:39,001] pudli/INFO/invokust.aws_lambda.lambda_load_test: Invocation complete. Requests (errors): 224 (120), execution time: 15066, sleeping: 0
[2020-06-28 19:58:40,116] pudli/INFO/root: threads: 1, rpm: 795, run_time: 18, requests_total: 224, request_fail_ratio: 0.5357142857142857, invocation_error_ratio: 0.0
[2020-06-28 19:58:40,117] pudli/ERROR/root: Error limit reached, invocation error ratio: 0.0, request fail ratio: 0.5357142857142857
[2020-06-28 19:58:40,117] pudli/INFO/root: Waiting for threads to exit...
[2020-06-28 19:58:54,086] pudli/INFO/invokust.aws_lambda.lambda_load_test: Invocation complete. Requests (errors): 242 (131), execution time: 15052, sleeping: 0
[2020-06-28 19:58:54,086] pudli/INFO/root: thread finished
[2020-06-28 19:58:54,142] pudli/INFO/root: Aggregated results: {"requests": {"GET_/": {"median_response_time": 92.0, "total_rps": 7.18569301694931, "avg_response_time": 91.08271769409947, "max_response_time": 114.66264724731445, "min_response_time": 84.4886302947998, "response_times": {"histogram": [85, 45, 4, 6, 7, 47, 11, 0, 0, 10], "bins": [84.0, 86.6, 89.2, 91.8, 94.4, 97.0, 99.6, 102.2, 104.8, 107.4, 110.0]}, "total_rpm": 431.1415810169586, "num_requests": 215}, "POST_/post": {"median_response_time": 150.0, "total_rps": 8.38878329746517, "avg_response_time": 157.73737294831653, "max_response_time": 1087.4686241149902, "min_response_time": 142.15636253356934, "response_times": {"histogram": [247, 0, 0, 1, 2, 0, 0, 0, 0, 1], "bins": [140.0, 236.0, 332.0, 428.0, 524.0, 620.0, 716.0, 812.0, 908.0, 1004.0, 1100.0]}, "total_rpm": 503.32699784791026, "num_requests": 251}}, "failures": {"POST_/post": {"method": "POST", "name": "/post", "error": "HTTPError('404 Client Error: Not Found for url: https://example.com/post',)", "occurrences": 251}}, "num_requests": 466, "num_requests_fail": 251, "total_lambda_execution_time": 30118, "lambda_invocations": 2, "approximate_cost": 6.3008e-05, "request_fail_ratio": 0.5386266094420601, "invocation_error_ratio": 0.0, "locust_settings": {"locustfile": "locustfile_example.py", "host": "https://example.com", "num_users": 2, "hatch_rate": 10, "run_time": "15s"}, "lambda_function_name": "lambda_locust", "threads": 1, "ramp_time": 0, "time_limit": 15}
[2020-06-28 19:58:54,142] pudli/INFO/root: ===========================================================================================================================
[2020-06-28 19:58:54,143] pudli/INFO/root: TYPE    NAME                                                #REQUESTS    MEDIAN   AVERAGE       MIN       MAX  #REQS/SEC
Scratch
[2020-06-28 19:58:54,143] pudli/INFO/root: ===========================================================================================================================
[2020-06-28 19:58:54,143] pudli/INFO/root: GET     /                                                         215      92.0     91.08     84.49    114.66       7.19
[2020-06-28 19:58:54,144] pudli/INFO/root: POST    /post                                                     251     150.0    157.74    142.16   1087.47       8.39
[2020-06-28 19:58:54,144] pudli/INFO/root: Exiting...
```
