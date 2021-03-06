# AWS Lambda version of snow-grafana-proxy

This version is really simplified, the configuration is hard-coded so you have to build your own Lambda package before publishing/using it. If you have any interest in making it more comfortable - let me know or (preferable) create a merge request with appropriate changes. If your ServiceNow instance replies slowly to API requqests  AWS Lambda may not be the best choice for you - **you'll pay for execution time wich includes the time you'll wait for SerciceNow replies**

If you're interested in my lesson learned from this subproject, you may want to check my [blog post](https://wordpress.com/post/funinit.wordpress.com/692)

## How to make it running
### Update configuration 
On top of `lambda-snow-grafana-proxy.py` you'll see:
```
queries={ "get_my_incidents": 
   { "table": "incident", 
     "snowFilter": "sysparam_query=active=true^incident_stateNOT IN 6,7,8", 
     "attributes": [ 
	{ "name": "assigned_to", "displayName": "Assigned to", "interpreter": "object_attr_by_link", "interpreterParams": { "linkAttribute": "name", "default": "FailedToGetName" } } , 
        { "name": "number", "displayName": "Number", "interpreter": "none" } 
     ] 
   } 
}

snowAuth=(("admin","pass"))
snowUrl="https://dev67310.service-now.com/"
```
queries is a dictionary defining query names - each query will be displayed as separate option available for grafana as a search. Configuration follows the same scheme as for the full version - please check docs there or [my blog posts about grafana-snow integration](https://funinit.wordpress.com/tag/servicenow/)[1,2]. 
snowAuth and snowUrl are pretty much self-explanatory, those are ServiceNow user credentials used to connect to snow and instance address.
### Test locally
Simply execute: ''' ./lambda-snow-grafana-proxy.py''' if you have requirements installed it should print JSON replies from service-now. Query may take some time... from my experience it may take some time to get ServiceNow reply. You can measure this value to configure appropriate timeout for query_post lambda function.
## Build deployment package
I recommend building AWS Lambda package with virtualenv, a commands I used are:
```
git clone snow-grafana-proxy
cd snow-grafana-proxy/aws-lambda
virtualenv -p /usr/bin/python36 venv
source ./venv/bin/activate
pip install logging
pip install requests
pip install json
cd venv/lib/python3.6/site-packages/
zip -r9 ~/lambda-snow-grafana-proxy.zip .
cd -
zip -g ~/lambda-snow-grafana-proxy.zip lambda-snow-grafana-proxy.py 
```
If everything finished successfully you should have `lambda-snow-grafana-proxy.zip` package in your $HOME directory. If this fails you may check [documentation provided by AWS](https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example-deployment-pkg.html#with-s3-example-deployment-pkg-python)
## Configure AWS services
### Creating AWS Lambda functions
You need to [install & configure awscli](https://aws.amazon.com/documentation/cli/) and have "basicLambdaRole" created first.
```
cd ~ 
aws lambda create-function --function-name search_post --runtime python3.6 --role arn:aws:iam::039238228955:role/basicLambdaRole --handler lambda-snow-grafana-proxy.search_post --zip-file fileb://lambda-snow-grafana-proxy.zip
aws lambda create-function --function-name query_post --timeout 30 --runtime python3.6 --role arn:aws:iam::039238228955:role/basicLambdaRole --handler lambda-snow-grafana-proxy.query_post --zip-file fileb://lambda-snow-grafana-proxy.zip
```
### Test with dummy invoke
```
#aws lambda invoke --function-name search_post out
$LATEST	200
#cat out
["get_my_incidents"]
#aws lambda invoke --function-name query_post --payload '{"targets": [ {"target": "get_my_incidents" } ] }' out
$LATEST	200
#cat out
[{"columns": [{"text": "Assigned to", "type": "string"}, {"text": "Number", "type": "string"}], "rows": [["Charlie Whitherspoon", "INC0000001"], ["Howard Johnson", "INC0000002"], ["Beth Anglin", "INC0000003"], ["Bud Richman", "INC0000004"], ["Bud Richman", "INC0000005"], ["Howard Johnson", "INC0000006"], ["David Loo", "INC0000007"], ["ITIL User", "INC0000008"], ["David Loo", "INC0000009"], ["Don Goodliffe", "INC0000010"], ["ITIL User", "INC0000011"], ["David Loo", "INC0000012"], ["David Loo", "INC0000013"], ["Bud Richman", "INC0000014"], ["Don Goodliffe", "INC0000015"], ["ITIL User", "INC0000016"], ["Fred Luddy", "INC0000017"], ["ITIL User", "INC0000018"], ["Bud Richman", "INC0000019"], ["ITIL User", "INC0000020"], ["Beth Anglin", "INC0000021"], ["ITIL User", "INC0000024"], ["ITIL User", "INC0000025"], ["Don Goodliffe", "INC0000026"], ["ITIL User", "INC0000027"], ["Don Goodliffe", "INC0000028"], ["Don Goodliffe", "INC0000029"], ["David Loo", "INC0000030"], ["David Loo", "INC0000031"], ["David Loo", "INC0000032"], ["Don Goodliffe", "INC0000033"], ["David Loo", "INC0000034"], ["Luke Wilson", "INC0000035"], ["Luke Wilson", "INC0000036"], ["Howard Johnson", "INC0000037"], ["Luke Wilson", "INC0000038"], ["FailedToGetName", "INC0000039"], ["ITIL User", "INC0000040"], ["ITIL User", "INC0000041"], ["Fred Luddy", "INC0000044"], ["FailedToGetName", "INC0000046"], ["Beth Anglin", "INC0000047"], ["ITIL User", "INC0000048"], ["Don Goodliffe", "INC0000049"], ["Beth Anglin", "INC0000050"], ["Don Goodliffe", "INC0000051"], ["Fred Luddy", "INC0000052"], ["Beth Anglin", "INC0000053"], ["FailedToGetName", "INC0000054"], ["Beth Anglin", "INC0000055"], ["FailedToGetName", "INC0000057"], ["FailedToGetName", "INC0000058"], ["FailedToGetName", "INC0000059"], ["David Loo", "INC0000060"]], "type": "table"}]
```
### Configure AWS API Gateway
It's really explained [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-with-lambda-integration.html), but if you'd like short version of what I did (aws description available [here](https://docs.aws.amazon.com/lambda/latest/dg/with-on-demand-https-example-configure-event-source.html):
```
#aws apigateway create-rest-api --name snowGrafanaProxy
HEADER	1533150017	749frzsiul	snowGrafanaProxy
TYPES	EDGE
#aws apigateway get-resources --rest-api-id 749frzsiul 
ITEMS	3oqa1jtsrj
#aws apigateway create-resource --rest-api-id 749frzsiul --parent-id 3oqa1jtsrj --path-part query
bkcpav	3oqa1jtsrj	/query	query
#aws lambda  list-functions --output json  | grep FunctionArn
            "FunctionArn": "arn:aws:lambda:us-west-2:039238228955:function:add", 
            "FunctionArn": "arn:aws:lambda:us-west-2:039238228955:function:query_post", 
            "FunctionArn": "arn:aws:lambda:us-west-2:039238228955:function:search_post", 
#aws apigateway put-integration --rest-api-id 749frzsiul --resource-id bkcpav --http-method POST --type AWS  --integration-http-method POST --uri arn:aws:apigateway:us-west-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-west-2:039238228955:function:query_post/invocations
bkcpav	POST	WHEN_NO_MATCH	29000	AWS	arn:aws:apigateway:us-west-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-west-2:039238228955:function:query_post/invocations
#aws apigateway put-method-response --rest-api-id 749frzsiul --resource-id bkcpav --http-method POST --status-code 200 --response-models "{\"application/json\": \"Empty\"}"
200
RESPONSEMODELS	Empty
# aws apigateway put-integration-response --rest-api-id 749frzsiul --resource-id bkcpav --http-method POST --status-code 200 --response-templates "{\"application/json\": \"\"}"
200
RESPONSETEMPLATES	None
# aws apigateway create-deployment --rest-api-id 749frzsiul --stage-name prod
1533152988	s80uq7
# aws lambda add-permission --function-name query_post --statement-id permission-for-api --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn "arn:aws:execute-api:us-west-2:039238228955:749frzsiul/*/POST/query"
{"Sid":"permission-for-api","Effect":"Allow","Principal":{"Service":"apigateway.amazonaws.com"},"Action":"lambda:InvokeFunction","Resource":"arn:aws:lambda:us-west-2:039238228955:function:query_post","Condition":{"ArnLike":{"AWS:SourceArn":"arn:aws:execute-api:us-west-2:039238228955:749frzsiul/*/POST/query"}}}
```
## Test 
As you can see below I've added time around that a few times I've got timeouts testing on ServiceNow developer instance. If your production ServiceNow performance is similar then... you may have similar issues

```
# time curl -X POST -d '{"targets": [ {"target": "get_my_incidents" } ] }' https://749frzsiul.execute-api.us-west-2.amazonaws.com/prod/query
[{"columns": [{"text": "Assigned to", "type": "string"}, {"text": "Number", "type": "string"}], "rows": [["Charlie Whitherspoon", "INC0000001"], ["Howard Johnson", "INC0000002"], ["Beth Anglin", "INC0000003"], ["Bud Richman", "INC0000004"], ["Bud Richman", "INC0000005"], ["Howard Johnson", "INC0000006"], ["David Loo", "INC0000007"], ["ITIL User", "INC0000008"], ["David Loo", "INC0000009"], ["Don Goodliffe", "INC0000010"], ["ITIL User", "INC0000011"], ["David Loo", "INC0000012"], ["David Loo", "INC0000013"], ["Bud Richman", "INC0000014"], ["Don Goodliffe", "INC0000015"], ["ITIL User", "INC0000016"], ["Fred Luddy", "INC0000017"], ["ITIL User", "INC0000018"], ["Bud Richman", "INC0000019"], ["ITIL User", "INC0000020"], ["Beth Anglin", "INC0000021"], ["ITIL User", "INC0000024"], ["ITIL User", "INC0000025"], ["Don Goodliffe", "INC0000026"], ["ITIL User", "INC0000027"], ["Don Goodliffe", "INC0000028"], ["Don Goodliffe", "INC0000029"], ["David Loo", "INC0000030"], ["David Loo", "INC0000031"], ["David Loo", "INC0000032"], ["Don Goodliffe", "INC0000033"], ["David Loo", "INC0000034"], ["Luke Wilson", "INC0000035"], ["Luke Wilson", "INC0000036"], ["Howard Johnson", "INC0000037"], ["Luke Wilson", "INC0000038"], ["FailedToGetName", "INC0000039"], ["ITIL User", "INC0000040"], ["ITIL User", "INC0000041"], ["Fred Luddy", "INC0000044"], ["FailedToGetName", "INC0000046"], ["Beth Anglin", "INC0000047"], ["ITIL User", "INC0000048"], ["Don Goodliffe", "INC0000049"], ["Beth Anglin", "INC0000050"], ["Don Goodliffe", "INC0000051"], ["Fred Luddy", "INC0000052"], ["Beth Anglin", "INC0000053"], ["FailedToGetName", "INC0000054"], ["Beth Anglin", "INC0000055"], ["FailedToGetName", "INC0000057"], ["FailedToGetName", "INC0000058"], ["FailedToGetName", "INC0000059"], ["David Loo", "INC0000060"]], "type": "table"}]
real	0m29.807s
user	0m0.098s
sys	0m0.086s

```

# List of links
[1] https://funinit.wordpress.com/2018/07/20/integration-for-servicenow-table-api-and-grafana/ \
[2] https://funinit.wordpress.com/2018/02/20/simple-integration-of-servicenow-and-grafana/ \
[3] https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example-deployment-pkg.html#with-s3-example-deployment-pkg-python \
[4] https://aws.amazon.com/documentation/cli/ \
[5] https://docs.aws.amazon.com/lambda/latest/dg/with-on-demand-https-example-configure-event-source.html
