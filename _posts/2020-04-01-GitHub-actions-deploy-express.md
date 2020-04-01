---
layout: post
title:  "Deploying an API built with Express.js to AWS API Gateway using GitHub Actions"
listed: true
---

In this post, I'll be describing the process of automatically deploying an API built with [Express.js](https://expressjs.com/) to AWS [API Gateway](https://aws.amazon.com/api-gateway/) using [claudia.js](https://claudiajs.com/) and GitHub actions.

## Setting up your AWS credentials

I'll assume you have you AWS ID and secret set up on your machine. In case you don't, take a look [here](https://claudiajs.com/tutorials/installing.html#configuring-access-credentials). 

I recommend creating a profile with an user that has only the permissions required by `claudia` (`AWSLambdaFullAccess`, `IAMFullAccess` and `AmazonAPIGatewayAdministrator`).

## Setting up `claudia`

In case you are not familiar with it, [claudia.js](https://claudiajs.com/) is a open-source deployment tool aimed to make the process of deploying a Node.js project to AWS Lambda easier. 

I'll summarize what we will need to configure, but if you run into any issues check their [official documentation](https://claudiajs.com/documentation.html).

First, go ahead and install it globally:

```
npm install claudia -g
```

### Updating your project

If you are developing using Express, you probably have something among this lines in your `app.js`:

``` js
'use strict'

var express = require('express');
var app = express();

app.get('/', function (req, res) {
  res.send('Hello World!');
});

app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
});
```

In order to use `claudia`, we'll have to export `app` instead of start listening. You should have something like this instead:

``` js
'use strict'

var express = require('express');
var app = express();

app.get('/', function (req, res) {
  res.send('Hello World!');
});

module.exports = app;
```

This will prevent you from running you API locally. If you want to, you can create a file called `app.local.js` with the following content:

``` js
'use strict'

var app = require('./app');

app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
});
```

And start your server locally using it.

### Creating the proxy

In order for your app to run on Lambda, you'll have to use a proxy. To create it, simply run

```
claudia generate-serverless-express-proxy --express-module app
```

This should create a `lambda.js` file in your project's root folder. Add it to source control.

### Creating the API

Run the following command to deploy your API to API Gateway:

```
claudia create --version [version] --config api-config.json --region [region] --handler lambda.handler --deploy-proxy-api --name [name] --profile [profile]
```

Replacing the values between square brackets for the appropriate value.

Parameter | Description
--------- | -----------
version   | Stage of your API (e.g. dev, prod)
region    | AWS region to deploy to (e.g `us-east-1`)
name      | The name of the API to be created on API Gateway
profile   | The profile you defined in your `.aws/credentials` file (e.g. `devops-credentials`)

A file named `api-config.json` should have been created. Add it to source control.

### Setting up your secrets on GitHub

In a web browser, navigate to you repository page and then to Settings > Secrets.

In here, add a key called `AWS_ACCESS_KEY_ID` and another called `AWS_SECRET_ACCESS_KEY` with the appropriate values. Also add keys for any other environment variables you want passed to your API (e.g. JWT secret, database connection string, etc.)

### Creating GitHub Actions

Back to your code editor, navigate to .github\workflows (create it if it doesn't exist). Create a file called `deploy.yml`.

Add the following content:

```yml
name: Send to AWS (dev)

# This action runs when we push to dev
on:
  push:
    branches:
    - dev

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v1
    # This step configures the AWS credentials used by claudia
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        # Replace this with the region you want to deploy to
        aws-region: us-east-1
    # This step converts the variables you defined as GitHub secrets to a JSON
    # This JSON will later be loaded by claudia and passed onto the API
    - name: Create environment JSON
      id: create-env
      uses: schdck/create-env-json@v1
      with:
        file-name: "./env.dev.json"
        # Declare here the variables you want to be passed to your API
        STAGE: "DEV"
        JWT_KEY: ${{ secrets.JWT_SECRET_DEV }}
        MONGO_URL: ${{ secrets.MONGO_URL_DEV }}
    # This step will install claudia globally and build/test your code
    - name: Use Node.js 12.x
      uses: actions/setup-node@v1
      with:
        node-version: '12.x'
    - name: npm install, build, and test
      run: |
        npm install
        npm install -g claudia
        npm run build --if-present
        npm test
      env:
        CI: true
    # This step will actually deploy your files to API Gateway
    - name: Send files to AWS
      run: claudia set-version --version dev --config api-config.json --set-env-from-json ${{ steps.create-env.outputs.full-path }}
```

If you want, you can create a `release.yml` that is run when pushes are made to `master`. Just remember to change the `--version` flag on the last step of the Action and the environment variables you are loading.

The reason I had to use an intermediate Action to convert the environment variables to JSON was that my environment variable's values contained commas (`,`), so I could not pass them using `claudia`'s `--set-env`. If this is not your case, you can remove that step and modify the last one to pass your variables directly in the command line.

---

That's it for today. As always, if you have any questions or feedback, please drop me a line 🙂