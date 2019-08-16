```
python3 -m venv ~/vscode/aws_arb_dash/.envdash
source ~/vscode/aws_arb_dash/.envdash/bin/activate
pip install -r requirements.txt
```

https://blog.apcelent.com/deploy-flask-aws-lambda.html

zappa deploy dev
zappa update dev
## 02 Dependencies layer
We need to provide all the Python libraries beyond the builtin ones that our project will use.
Building package libs should be done in the lambci docker container to match wth lambda executor environment. We will use pip to directly write the libraries to a specific directory, important for Python to locate them.

Run from the project root:
```Shell
docker run --rm -it -v ${PWD}/layers/02_dependencies:/var/task lambci/lambda:build-python3.6 bash
# From the docker container, install the python packages
# rm -r python
# pip install -r requirements.txt --target python/lib/python3.6/site-packages/ --no-deps --ignore-installed
pip install -r requirements.txt --target python/lib/python3.6/site-packages/ --no-deps
exit

```
### Make a zip file for the layer
```Shell
rm layers/zips/02_dependencies.zip
pushd layers/02_dependencies
zip -r ../zips/02_dependencies.zip python
popd
aws s3 sync layers/zips s3://arbitrator-store/lambda-dash/layers --delete
aws lambda publish-layer-version --layer-name arb_dash_dependencies --description "dash dependencies layer" --content S3Bucket=arbitrator-store,S3Key=lambda-dash/layers/02_dependencies.zip --compatible-runtimes python3.6

```





## 03 Rain model layer
The files for this layer already exist on the `layers/03_rain_model/` directory, so noDocker hijinks needed here.
You need to put your api keys into the file called `keys.csv.template` and rename it to `keys.csv`.

Despite the name of the layer, I'll use it to push the API keys csv to your layer in addition to the model object.
```
rm layers/zips/03_rain_model.zip
pushd layers/03_rain_model/
zip -r ../zips/03_rain_model.zip .
popd
```
### Push the zip file to s3 for building the layer
```
aws s3 sync layers/zips s3://ds-upskill-bucket/lambdas/layers --delete
```
### Build/update layer 03_rain_model
```
aws lambda publish-layer-version --layer-name weather_03_model --description "model layer" --content S3Bucket=ds-upskill-bucket,S3Key=lambdas/layers/03_rain_model.zip --compatible-runtimes python3.6
```

## AWS Console
- Confirm that layer zip files exist in S3.
- go to the Lambdas section
  - Confirm that new layers have been created.

## DynamoBD 
In AWS go to the DynamoDB section
- Click "Create Table"
  - Table name: weather_lambda_query_log
  - Partition key: city (string)
  - Sort key: timestamp (number)
  - Click create

## Back to the Lambda
Open your lambda.
### Add the layers
We don't need all the layers right away, but will add them all now, then we're done. 
Beneath your Lambda name, click `Layers` and add the following layers:
1. Preconfigured Python 3.6 numpy/scipy + OS deps layer
2. Our libraries dependencies weather_02_dependencies
3. Our model and keys file layer weather_02_model

## Add geolocation
Copy paste the code from `weather_02.py` into the code window and test.

## Add weather query
Copy paste the code from `weather_03.py` into the code window and test.

## Add model prediction
Copy paste the code from `weather_04.py` into the code window and test.

## Add writing to DynamoDB and S3
Copy paste the code from `weather_05.py` into the code window and test.