---
layout: post
date: 2018-04-26T02:25:30Z
title: "Dynamic Image resizing using Amazon S3, AWS lambda, Amazon API Gateway and Amazon CloudFront"
description: Images often account for most of website’s content; In fact,
    according to the HTTP Archive, images constitute more than 60% a website’s total bandwidth. 
    
    When Google's PageSpeed Insights Tool detects that the images on a page are not optimized,  recommends serving appropriately scaled images (in addition to using proper image format and compression to reduce image size), which are as close as possible to the intended size in the browser, as a key factor in improving your page loading speed; and as the result improving your SEO.
active: true
mainFeatured: false
image: https://d3u9etne7s6j1g.cloudfront.net/blog/img/600x600/dynamic-image-resizing-aws-services-architecture.png
cropImage: https://d3u9etne7s6j1g.cloudfront.net/blog/img/600x200/dynamic-image-resizing-aws-services-architecture.png
img-credit-url: 
img-credit-desc: 
categories: tech
permalink: /:categories/:title/
tags:
  - Amazon S3
  - AWS lambda
  - Amazon API Gateway
  - Amazon CloudFront
author-avatar: https://d3u9etne7s6j1g.cloudfront.net/base/img/60x60/author-avatar-default.jpg
author-name: Moses
author-title: 
---

Images often account for most of website’s content; In fact, according to the [HTTP Archive](http://httparchive.org/){:target="_blank"}, images constitute more than 60% a website’s total bandwidth. 

When Google's PageSpeed Insights Tool detects that the images on a page are not optimized,  recommends serving appropriately scaled images (in addition to using proper image format and compression to reduce image size), which are as close as possible to the intended size in the browser, as a key factor in improving your page loading speed; and as the result improving your SEO.

Serving large image assets only to have the browser resize them is not only an overhead of shipping unnecessary pixels, but also consumes extra CPU resources.

One way of achieving this goal is to pre-process the images to every possible sizes which is not an ideal solution; the other good solution, instead, is to generate images dynamically when there’s a demand and make them available for future use.

In this post we will cover in depth how we can dynamically resize images using AWS Lambda, API Gateway and Amazon S3; and serve them through Amazon CloudFront CDN.

##### Workflow in brief

Looking at the diagram above :

- Steps i, ii and iii : The ultimate goal of these steps is to upload original images to an S3 bucket where CloudFront can use as an origin server.

  **Note:** These steps can be done in isolation from the rest of the automation process.

  - Step i:	A Developer uploads images to S3 bucket **bucket-site-asset-original**.

    **Note:** This can be done using AWS S3 console to manually upload the images, but it’s best to automate that part of the process using CI. For information about automating using Travis CI, see [Uploading a specific folder of your project from your repository to amazon s3 bucket using Travis CI](/Uploading-a-specific-folder-of-your-project-from-your-repository-to-amazon-s3-bucket-using-Travis-CI/)

  - Step ii:	Lambda function **LambdaCopyBucketOriginalToOriginServer** gets  
triggered by S3 bucket **bucket-site-asset-original** on **ObjectCreated** and **ObjectRemoved** event types.

  - Step iii: Lambda function **LambdaCopyBucketOriginalToOriginServer** gets
 executed and copies object over into or removes from **bucket-site-asset-origin-server** depending on the event type by object key.
- Step 1 : A user requests a resized image from CloudFront distribution
- Step 2 : The image is found in the CloudFront distribution; the work flow completes
- Step 3 : The requested image is not found in the CloudFront, CloudFront requests to get it from origin server **bucket-site-asset-origin-server**
- Step 4 : The requested image is found in the origin server **bucket-site-asset-origin-server**; it returns it to CloudFront, CloudFront returns the request to the user, also CloudFront caches it in all regional edge locations, 
- Step 5 : The requested image in not found in the origin server **bucket-site-asset-origin-server**, the origin server is set up to host as static website and in that we have a routing rule for resources which are not found (404) to respond with 307 redirect to the API Gateway **APIGatewayResizeImage** endpoint url via CloudFront, which we’ll see in detail later in this post.
- Step 6 : The user browser makes a request to the API Gateway
- Step 7 : The Lambda function **LambdaResizeImage** is triggered by the API Gateway
- Step 8 : The Lambda function **LambdaResizeImage** gets the original image from S3 bucket **bucket-site-asset-origin-server**
- Step 9 : The S3 bucket **bucket-site-asset-origin-server** returns the orignal image
- Step 10 : After the Lambda function **LambdaResizeImage** completes its resizing operations, it puts the processed image into S3 bucket **bucket-site-asset-origin-server**
- Step 11 : The Lambda function **LambdaResizeImage** invalidates the response from CloudFront cache for the original request which is **307 redirect** for the next same request to pass through
- Step 12 : The Lambda function **LambdaResizeImage** returns the requested resized image to the API Gateway **APIGatewayResizeImage**
- Step 13 : The API Gateway returns the requested image to the User.

Now let’s dive into the implementation detail…

##### Prerequisites

1. Have an amazon AWS account
2. Have a basic knowledge of node.js.

This post is divided into two parts, [Part I](#partI) and [Part II](#partII); 

Part I explains the steps involved in the first part of the flow shown in the above diagram above with i, ii and ii which are:

- [creating S3 buckets](#create-s3-buckets), 
- [creating IAM policies](#create-iam-policies),
- [creating roles and attaching policies](#create-roles), 
- [creating a Lambda function](#create-lambda-function). 

If you already are familiar with these, skim [Part I](#partI) and go to [Part II](#partII) which explain the steps involved in creating images dynamically which are:

- [Creating CloudFront Distribution](#create-cf-dist)
- [Creating an image resizing Lambda function](#create-resize-lambda)
- [Creating an API Gateway](#creat-api-gateway)

<div class="anchor" id="partI"></div> 

### Part I
<div class="anchor" id="create-s3-buckets"></div> 

#### Step 1: Create S3 buckets 

In this step, we will create two S3 buckets :

1. **bucket-site-asset-original** : to be used as a storage for our original images and we’ll keep it in sync with our repository.
2. **bucket-site-asset-origin-server** : to be used as a storage for original images and resized images; as well as to be used as Origin server by the CloudFront Distribution.

 <div class="NOTE alert">
 <p><i class="fa fa-info-circle"></i> Note</p>
 <p>You can use the first bucket as the storage and origin server for CloudFront if you prefer the bucket not to be in sync with your repository; but I recommend using two buckets.</p>
</div>


Follow these steps to create an S3 bucket:

- Sign in to your AWS Management Console and open the S3 console at [https://console.aws.amazon.com/s3](https://console.aws.amazon.com/s3){:target="_blank"}
- Click on the **Create bucket** button.
- On the pop-up window, type in a name of the bucket.
- Click **Next** button through the wizard leaving the default settings as is.

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>You need to repeat the above steps one more time to create the second bucket.</p>
</div>

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/s3-bucket-create.png  "s3 bucket create"){:width="80%"}

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/s3-bucket-created.png  "s3 bucket created"){:width="80%"}

<div class="anchor" id="create-lambda-function"></div> 

#### Step 2: Create a Lambda function

Before we go ahead and create the Lambda function, we need to first cover a very important step which is AWS Identity and Access Management (IAM) service role.

For an AWS service to access resources from another service, it must be configured to assume a role with permissions it requires to perform actions on those resources  on behalf of your account (in this case, for example, a Lambda function to access an S3 bucket resource to perform actions on it, like read object, remove object, list bucket...).

In AWS service environment, permissions are defined in the form of policies, which later get attached to a role, which again can be used to configure  **Execution role** for a Lambda function.

<div class="anchor" id="create-iam-policies"></div> 

##### Step 2.1: Create policy

So now, let’s first create a policy which, in AWS terms, is called **Customer Managed** policy, and then attach it to the role which we’ll be creating next.

Follow these steps to create a policy:

- Sign in to your AWS Management Console and open the IAM console at [https://console.aws.amazon.com/iam](https://console.aws.amazon.com/iam){:target="_blank"}
- Select **Policies** from the navigation pane and then click on the **Create policy** button.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/aws-policy-create.png  "aws policy create"){:width="80%"}

- On the next page, on **Visual editor** tab, 
  - For **Service**: 
    - Click on **Choose a service** link, 
    - From the list below, find and select S3.
  - For **Actions**: 
    - Click on **Select actions** link,
    - Expand **List** under **Access level** and select **ListBucket**
    - Expand **Read** and select **GetObject** and **GetObjectAcl**
  - For **Resources**:
    - Select **Specific** and click on the **Add ARN** link below corresponding to **bucket**
    - On the pop-up window, enter the ARN for the first bucket you created in the previous step in the **Specify ARN for bucket** text box and click on the **Add** button.
    - Click on the **Add ARN** link corresponding to **object**
    - On the pop-up window, enter the ARN for the bucket followed by **/img/\***, as, **arn:aws:S3:::bucket-site-asset-original/img/\*** and click on the **Add** button (we add the **/img** path because the bucket could hold other site assets which we may not want the Lambda function to have permission to access)
<div class="TIP alert">
    <p><i class="fa fa-lightbulb-o"></i> Tip</p>
    <p>You can copy Bucket ARN by selecting the bucket name and from
the pop-up window, click on the <strong>Copy Bucket ARN</strong></p>
</div>
<!-- 
  ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/s3-bucket-copy-ARN.png  "s3 bucket copy ARN"){:width="80%"} -->

  ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/IAM-create-policy.png  "IAM create policy"){:width="80%"}

Now, let’s add additional permission for the Lambda function to have access to the objects on the second bucket **bucket-site-asset-origin-server**

- On the same page, click on the **Add additional permissions** link at the bottom right corner
  - For **Service**: 
    - Click on **Choose a service** link, 
    - From the list below, find and select S3.
  - For **Actions**: 
    - Click on **Select actions** link,
    - Expand **List** under **Access level** and select **ListBucket**
    - Expand **Write** and select **DeleteObject** and **PutObject**
    - Expand **Permissions management** and select **PutObjectAcl**
  - For **Resources**:
    - Select **Specific** and click on the **Add ARN** link below corresponding to **bucket**
    - On the pop-up window, enter the ARN for the second bucket you created in the previous step in the **Specify ARN for bucket** text box and click on the **Add** button.
    - Click on the **Add ARN** link corresponding to **object**
    - On the pop-up window, enter the ARN for the bucket followed by **/img/\***, as, **arn:aws:S3:::bucket-site-asset-origin-server/img/\*** and click on the **Add** button  
- Click on the **Review policy** button at the bottom right corner
- On the next page, enter a name and description and click on the **Create policy** button

  ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/IAM-name-policy.png  "IAM name policy"){:width="80%"}

<div class="anchor" id="create-roles"></div> 

##### Step 2.2: Create role

  Follow these steps to create a role:

- Sign in to your AWS Management Console and open the IAM console at [https://console.aws.amazon.com/iam](https://console.aws.amazon.com/iam){:target="_blakk"}
- Select **Roles** from the navigation pane and then click on the **Create role** button.
- Choose the AWS Service role type **AWS service**, and then choose the service that you want to allow to assume this role **Lambda**, and then click on the **Next: Permissions** button

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/IAM-create-role-select-type-service.png  "IAM create role select type service"){:width="80%"}


- Search and select the policy you created in the previous step 
- Search and select **CloudWatchFullAccess**, and then click on the **Next: Review** button
- On the next page, enter role name, and then click on the **Create role** button

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/AIM-create-role-wizard-end.png  "AIM create role wizard end"){:width="80%"}

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>We added the <strong>CloudWatchFullAccess</strong> policy to the role so that logs can be written into CloudWatch from the Lambda function.</p>
</div>

With the IAM piece covered, Now, we’re ready to create the Lambda function **LambdaCopyBucketOriginalToOriginServer**

##### Create a Lambda function

Follow these steps to create the Lambda function **LambdaCopyBucketOriginalToOriginServer** :

- Sign in to your AWS Management Console and open the IAM console at [https://console.aws.amazon.com/lambda](https://console.aws.amazon.com/lambda){:target="_blank"}
- Click on the **Create function** button at the top-right corner
- Select **Author from scratch**, and give it a name 
- For **Runtime** : select **Node.js 6.10**
- For **Role** : select **Choose an existing role**
- For **Existing role** : select the role you created in the previous step
- Click on the **Create function** button

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/Lambda-create-function.png  "Lambda create function"){:width="80%"}

- On the next page, from the left pane, click on S3
- Scroll down to the **Configure triggers** section
  - For **Bucket** : select the first bucket you created above
  - For **Event type** : select **Object Created (All)**
  - For **Prefix** : type in **img/**
  - Click on the **Add** button.

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>This covers the <strong>Object Created</strong> event type, we need to configure a trigger for <strong>Object Removed</strong> event type.</p>
</div>
 
- From the left pane, click on S3
  - For **Bucket** : select the first bucket you created above
  - For **Event type** : select **Object Removed (All)**
  - For **Prefix** : type in **img/** 
  - Click on the **Add** button.

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/Lambda-creat-trigger.png  "Lambda creat trigger"){:width="80%"}


- Click on the **Save** button at the top of the page.
- Click on the Lambda function box

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/Lambda-configure-function.png  "Lambda configure function"){:width="80%"}

- Scroll down to the **Function code** section, and then add the following code to the index.js file. 

<pre class="prettyprint linenums">
var AWS = require("aws-sdk");
 
exports.handler = (event, context, callback) => {
    const req = JSON.stringify(event);
    console.log(`Event  ${req} triggered`);
    var s3 = new AWS.S3();
    var sourceBucket =  process.env.source_bucket;
    var destinationBucket = process.env.dest_bucket;
    var objectKey = event.Records[0].s3.object.key;
    var copySource = encodeURI(sourceBucket + "/" + objectKey);
    var copyParams = { Bucket: destinationBucket, CopySource: copySource, Key: objectKey };
    var delParams = { Bucket: destinationBucket, Key: objectKey };
    var eventName = event.Records[0].eventName;
    if(eventName === "ObjectRemoved:Delete") {
        s3.deleteObject(delParams, function(err, data) {
            if (err) {
                console.log(err, err.stack);
            } else {
                console.log("S3 object delete successful.");
            }
        });
    } else if(eventName === "ObjectCreated:Put") {
        s3.copyObject(copyParams, function(err, data) {
            if (err) {
                console.log(err, err.stack);
            } else {
                console.log("S3 object copy successful.");
            }
        });
    }
};
</pre>

In the **Environment variables** section, enter these key-value pairs for the two buckets

- Key: **source_bucke**  Value: **[Add-the-Name-of-the-First-Bucket-you-created]**
- Key: **dest_bucket**  Value: **[Add-the-Name-of-the-Second-Bucket-you-created]**

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/Lambda-create-env-var.png  "Lambda create env var"){:width="80%"}

- Click on the **Save** button

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>At this stage, you can test this Lambda function independent of the rest of the workflow.</p>
</div>

To test, You can use the s3 console by going into the source bucket and create **img** folder, and then inside the **img** folder, you upload an image and see if it makes it to the destination folder; similarly remove an image from the source bucket and see if it’s removed from the destination bucket.

 ![alt text]({{ site.assetBaseUrl }}/blog/img/lambda-copy-s3-buckets.gif  "s3 lambda copy s3 buckets"){:width="100%"}

<div class="anchor" id="partII"></div> 

### Part II


In this part, we’ll look into the steps to accomplish the actual dynamic resizing of images. 
<div class="anchor" id="create-cf-dist"></div> 

#### Step 1: Create CloudFront Distribution

Before we create the CloudFront distribution, we need to go back to the S3 bucket **bucket-site-asset-origin-server** and do some configuration changes.

- Click on to the bucket from S3 console
On the next page, click on the **Properties** tab, and then click on the **Static website hosting** box
- Select **Use this bucket to host a website** option
- Enter index.html in the **Index document** text box
- Click on the **Save** button
- On the same page, click on the **Permissions** tab, and then click on the **Bucket Policy** box below
- Copy and paste the following code in the **Bucket policy editor** box to give only read permission to everyone.


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::bucket-site-asset-origin-server/*"
        }
    ]
}
```
<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>Replace <strong>bucket-site-asset-origin-server</strong> with the name you give for your origin server bucket</p>
</div>
- Click on the **Save** button

Now we’re ready to create the CloudFront Distribution.

Follow these steps to create the CloudFront Distribution :

- Sign in to your AWS Management Console and open the IAM console at [https://console.aws.amazon.com/cloudfront](https://console.aws.amazon.com/cloudfront){:target="_blank"}
- Click on the **Create Distribution** button
- On the next page, click on the **Get Started** button under the **Web** section
- On the next page, in the **Origin Domain Name** text box, enter the endpoint url for the bucket we set up to be hosted as a website

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/s3-bucket-endpoint-url.png  "s3 bucket endpoint url"){:width="80%"}

- Leave the rest of the default settings, and then click on the **Create Distribution** button at the bottom-right corner of the page

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>It takes several minutes before CloudFront deploys the newly created Distribution, wait the status to change to <strong>Deployed</strong> before you test.</p>
</div>

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/cloudfront-create-distribution.png  "cloudfront create distribution")

Once it’s deployed, you try accessing an image which is present in the origin server bucket using the CloudFront Distribution domain name, as **[your-cloudfront-distributionid].cloudfront.net/img/[image-name-you-uploaded]**
<div class="anchor" id="create-resize-lambda"></div> 

#### Step 2: Create the Lambda function **LambdaResizeImage**

The steps for creating this function is similar to the one explained in Part I; the difference would be Execution role, function code and Environment variables. Do not worry about adding a trigger to it at this point as we’ll be making the connection from the API Gateway configuration side which we’ll see in the next step.

Copy the following code and paste it into index.js
<pre class="prettyprint linenums">
'use strict';

const im = require('imagemagick');
const fs = require('fs');
var AWS = require('aws-sdk');
//AWS.config.update({region: 'us-east-1'});
const s3 = new AWS.S3();
var cloudfront = new AWS.CloudFront();

const BUCKET = process.env.BUCKET;
const ALLOWED_DIMENSIONS = new Set();
const ALLOWED_IMAGE_TYPES = new Set();
if (process.env.ALLOWED_DIMENSIONS) {
  const dimensions = process.env.ALLOWED_DIMENSIONS.split(/\s*,\s*/);
  dimensions.forEach((dimension) => ALLOWED_DIMENSIONS.add(dimension));
}

if (process.env.ALLOWED_IMAGE_TYPES) {
  const image_types = process.env.ALLOWED_IMAGE_TYPES.split(/\s*,\s*/);
  image_types.forEach((type) => ALLOWED_IMAGE_TYPES.add(type));
}

const invalidateCfObject = (key) => {
    return new Promise(function (resolve, reject) {
        
    const distId = process.env.CF_DistributionId; 
    var invalidationPath = key;
    console.log('invalidation path : ' + invalidationPath);  
    var callRef = Date.now();
    console.log('Date.now() : ' + Date.now() + ',  toString() :' 
        + callRef.toString()); 
    var params = {
      DistributionId: distId,
      InvalidationBatch: { 
        CallerReference: callRef.toString(),
        Paths: { 
            Quantity: 1,
            Items: [  '/'+invalidationPath  ]
        }
      }
    };
    cloudfront.createInvalidation(params, function(err, data) {
    if (err) {
        console.log(err, err.stack);
        return reject(err); }// an error occurred
    else     {
        console.log('invalidation result :' + JSON.stringify(data));
        resolve(data);          // successful response
    }
    });
    });
};

const resizeImage = (reqParams) => {
    return new Promise(function (resolve, reject) {
        if(!reqParams.base64Image) {
            const errMsg = 'Invalid resize request: no "base64Image" field supplied';
            console.log(errMsg);
            return reject(errMsg);
        }
        const tempFilePath = `/tmp/resized${(reqParams.imgType || '.png')}`;
        
        const params = {
            srcData: reqParams.base64Image,
            dstPath: tempFilePath,
            width: reqParams.width,
            height: reqParams.height
        };
        im.resize(params, function(err, resp) {
            if (err) {
                console.log(err);
                return reject(err);
            } 
            resolve(tempFilePath);
       });
    });
};


exports.handler = (event, context, callback) => {
    
    const bucketKey = event.params.querystring.key;
    
    const match = bucketKey.match(/((\d+)x(\d+))\/(.*)/);
    const imageTtype = bucketKey.match(/\.[0-9a-z]+$/i);
    
    const image_type = imageTtype[0];
    const dimensions = match[1];
    const width = parseInt(match[2], 10);
    const height = parseInt(match[3], 10);
    const originalKey = match[4];
    
    if(ALLOWED_DIMENSIONS.size > 0 && !ALLOWED_DIMENSIONS.has(dimensions)) {
        const dimErrorObj = {
        errorType : "Forbidden",
        status : 403,
        requestId : context.awsRequestId,
        message : "unsupported image dimensions."
        };
       callback(JSON.stringify(dimErrorObj));
       return;
    }
    
    if(ALLOWED_IMAGE_TYPES.size > 0 && !ALLOWED_IMAGE_TYPES.has(image_type)) {
        const typeErrorObj = {
        errorType : "Forbidden",
        status : 403,
        requestId : context.awsRequestId,
        message : "unsupported image type."
        };
       callback(JSON.stringify(typeErrorObj));
       return;
    }

    const imagePath = 'img/' + originalKey;
   
    const bucketName = BUCKET; 
    
    s3.getObject({
                Bucket: bucketName, //orgBucketName, 
                Key: imagePath
              }).promise()
    .then(data => resizeImage({base64Image: new Buffer(data.Body), width: width, height: height, imgType: image_type}))
    
    .then(filePath => fs.readFileSync(filePath))
    .then(buffer => {
                    s3.putObject({
                    Bucket: bucketName, 
                    Key: bucketKey,
                    Body: buffer,
                    ContentType: 'image/png',
                    CacheControl: 'max-age=604800'
                }).promise();
                return buffer;
    })
    .then(buffer => {
            invalidateCfObject(bucketKey);
            return buffer;
    })
    .then(buffer => callback(null, buffer.toString('base64')))
    .catch(err => {
        console.log(err.toString());
        var errObj = {
            errorType : "Internal Server Error",
            status : 500,
            requestId : context.awsRequestId,
            message : err.toString()
        };
        callback(JSON.stringify(errObj));
    });
 };

</pre>

This Lambda function first extracts the image dimensions and image type from the **key** parameter, and then does validations for allowed dimensions and image type; If all valid, it gets the original image from the bucket and performs resizing operation as per the requested dimensions; after successful completion of the operation, it puts it in a in the bucket with the dimension as a folder name; and finally before it returns with the response, it invalidates the CloudFront for the first request key whose response was 307 redirect (at the time of this writing, CloudFront caches 307 redirect responses, but hopefully AWS will put some kind of mechanism soon by which developers can have control over caching behavior on 30* redirects).

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>In this function we used ImageMagick to resize images, Imagemagick is very powerful and we can extend this function to include it’s other cool features like crop an image, blur, despeckle, convert image formats...</p>
</div>

In **Environment variables** section, add these key-value pairs 

- Key: **ALLOWED_DIMENSIONS** Value: **100x100,150x100, 150x200, 200x100,200x200, 600x300**
- Key: **ALLOWED_IMAGE_TYPES** Value: **.png,.jpeg,.jpg**
- Key: **BUCKET** Value: **[Add-Your-Origin-Server-Bucket-Name]**
- Key: **CF_DistributionId** Value: **[Add-Your-Distribution-ID]**

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/lambda-image-resize-function.png  "lambda image resize function"){:width="80%"}

Creating the role with associated policies for this function which I will leave it for you as exercise.

<div class="TIP alert">
  <p><i class="fa fa-info-circle"></i> Tip</p>
  <p>For execution role, you need to first create a policy which gives required permission to access the Origin server bucket, and then attach it to the role along with <strong>CloudFrontFullAccessPolicy</strong> and <strong>CloudWatchFullAccessPolicy</strong></p>
</div>

<div class="anchor" id="creat-api-gateway"></div> 


#### Step 3: Create API Gateway

With this API Gateway, we expose our image resizing Lambda function to be invoked through a GET method on a public interface of an HTTP endpoint. 

Follow these steps to create an API Gateway :

- Sign in to your AWS Management Console and open the API Gateway console at [https://console.aws.amazon.com/apigateway](https://console.aws.amazon.com/apigateway){:target="_blank"}
- Click on the **Create API** button 
- On the next page, give it a name, and then click on the **Create API** button.
- On the next page, Click on the drop-down button **Actions** next to **Resources** 

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gw-create-method.png  "api gateway create method"){:width="80%"}

- Select **Get** from the drop-down list and click on the check mark icon next to it to save
- On the page provided, click on the the text box next to **Lambda Function** and press any key to list Lambda functions you have, and then select the one you created in the previous step to set up the integration.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-select-lambda-function.png  "api gateway select lambda function"){:width="80%"}

- Click on the **Save** button, and on the popup window, click on the **OK** button confirming that you give this api permission to invoke the Lambda function.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-get-method-execution.png  "api gateway get method execution"){:width="80%"}

In addition to being a front-end interface to our Lambda function, API Gateway can perform basic request input validation, map the payload from a method request to the corresponding integration request as required in our Lambda function and from integration response to the corresponding method response as expected by a client caller.

API Gateway makes use of a mapping template which is a script expressed in Velocity Template Language (VTL) and apply in to the payload to do the mapping.

Since our Lambda function expects a query string **key** value in its JSON structure, Let’s first configure request validation to make sure the request has **key** query string. To do that:

- Click on the **Method Request** link on the top-left box
- Under **Settings** section, click on the pencil icon next to **Request Validator** and select **Validate query string parameters and headers** from the list, and then click on the check mark to save it
- Expand the **URL Query String Parameters** and click on the **Add query string** link
- Add **key** in the text box provided and click on the check mark to save it
- Once saved, click on the check box below **Required** column.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-request-validator.png  "api gateway request validator"){:width="80%"}

Next, we need to add a mapping template on Integration Request to map the request to a JSON payload to our Lambda function.

Follow these steps to add a mapping template :

- Click on the **Integration Request** link on the top-right box
- On the next page, expand **Body Mapping Templates**
- Select **When there are no templates defined (recommended)**
- Click on the **Add mapping template** link below **Content-Type**
- Type in **application/json** in the text box, and then click on the check mark to save
- In the text area provided below, copy and paste the below mapping template, which converts all request parameters including path, querystring and header to a JSON payload.

```
#set($allParams = $input.params())
{
  "params" : {
    #foreach($type in $allParams.keySet())
    #set($params = $allParams.get($type))
    "$type" : {
      #foreach($paramName in $params.keySet())
      "$paramName" : "$util.escapeJavaScript($params.get($paramName))"
      #if($foreach.hasNext),#end
      #end
    }
    #if($foreach.hasNext),#end
    #end
  }
}
```

- Click on the **Save** button.

And finally, since our Lambda function, after successful resizing of an image,  responds with a base64-Encoded string of it and clients expect a binary data, we need to do some configuration on Integration response to do the conversion for 200 Method response status. But before that we need to make sure on the Method Response that we define all the HTTP status codes that we want our API Gateway to respond to the clients.

- Click on the **Method Response** link on the bottom-left box
- Expand the **200** HTTP Status which already defined as default, and Under **Response Headers** for **200**, click on the **Add Header** link
- Type in **Content-Type** and click on the check mark to save it
- Click on the **Add Header** link again, and type in **Cache-Control** and click on the check mark to save it

Now let’s add one more HTTP Status for error response

- Click on the **Add Response** link below
- Enter **403**, and then click on the check mark to save

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-method-response.png  "api gateway method response"){:width="80%"}

- Click on the **Method Execution** link at the top to go back to the previous page

Now we’re ready to configure the Integration response; 

- Click on the **Integration Response** link of the bottom-right box

For **200** Method response status: 

  - Expand the list item with **200** method response status
  - From the **Content handling** drop-down box, select **Convert to binary (if needed)**
  - Expand **Header Mappings** and click on the pencil icon corresponding **Cache-Control** and enter **'max-age=604801'** (with the quotes) in the **Mapping value** text box, and then click on the check mark to save it
  - Click on the the pencil icon corresponding **Content-Type** and enter **'image/png'** (with the quotes), and then click on the check mark to save it

For **403** Method response status: 

  - Expand the list item with **403** method response status
  - Enter **.*"status":403.\*** in the text box next to Lambda Error Regex 
  - Select **403** from **Method response status** drop-down list
  - Click on the **Save** button
  - Once saved, click on it to expand, and then on the **Body Mapping Template** section, click on the **Add mapping template link**
  - Enter **application/json** in the text box and click on the check mark to save
  - In the template box on the right, enter **$input.path('$.errorMessage')**
  - Click on the **Save** button at the bottom-right corner of the screen.
<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>Cache-Control defines who can cache responses and for how long; In this example, for <strong>200</strong> method response, we set it with a max-age of seven days  <strong>(Cache-Control: max-age=604801)</strong>, but it’s recommended that you set it to even a higher value for static assets like images which will not change in the future.</p>
</div>

Now let’s go ahead and deploy out API Gateway; 

- From **Actions** drop-down button, select **Deploy API** option
- On the pop-up screen, from the **Deployment stage** drop-down, select **[New Stage]** 
- Enter Stage name as **prod**
- Click on the **Deploy** button


![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-deploy-api.png  "api gateway deploy api"){:width="80%"}


![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/api-gateway-deploy.png  "api gateway deploy"){:width="80%"}

A final step we need to do before we give the entire flow a test is that to add a routing rule to the S3 bucket website to redirect 404 not found responses.

To do that : 

- Click on to the bucket from S3 console
- On the next page, click on the **Properties** tab, and then click on the **Static website hosting** box
- In the **Redirection rules (optional)** box, add the following, and then click on the **Save** button.


```xml
<RoutingRules>
  <RoutingRule>
    <Condition>
      <KeyPrefixEquals>img/</KeyPrefixEquals>
      <HttpErrorCodeReturnedEquals>404</HttpErrorCodeReturnedEquals>
    </Condition>
    <Redirect>
      <Protocol>https</Protocol>
      <HostName>[your-api-gateway-endpoint]</HostName>
      <ReplaceKeyPrefixWith>prod?key=img/</ReplaceKeyPrefixWith>
      <HttpRedirectCode>307</HttpRedirectCode>
    </Redirect>
  </RoutingRule>
</RoutingRules>
```

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/s3-bucket-website-routing-rule.png  "s3 bucket website routing rule"){:width="80%"}

<div class="NOTE alert">
  <p><i class="fa fa-info-circle"></i> Note</p>
  <p>Replace <strong>HostName</strong> entry with your API Gateway Invoke URL endpoint which you can copy it from the top of the page (excluding the stage name) by going to the API Gateway you just created and select <strong>Stages</strong> from the left pane and then click on the stage name <strong>prod</strong></p>
</div>

#### Test

From a test website, using the CloudFront Distribution, let’s reference the image we uploaded with a size dimensions to see the image resizing flow in action; and let’s also reference the same image without dimensions to access the original image and resize it in the browser to see the difference in bytes downloaded:

```
<img src="https://[your-distribusion-id].cloudfront.net/img/150x200/[your-upploaded-image-file-name]">
<img src="https://[your-distribusion-id].cloudfront.net/img/[your-upploaded-image-file-name]" width="150px">
```

##### Very first request



 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/test-image-resize.png  "test image resize 1st request"){:width="80%"}

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/fiddler-image-resize-1st-request.png  "fiddler image resize 1st request"){:width="80%"}

When a request for a resized image comes to CloudFront for the very first time, it makes a request for it from Origin server; since the resized image is not yet created, the Origin server responds with 404 (not found) error; which makes the condition we put in the Routing rule setup to be true, enforcing the response to be a 307 redirect to the API Gateway endpoint, which is also part of the routing rule. This response will reach the browser via CloudFront, and then the browser makes a request to the API Gateway which is our trigger point for the **ImageResizeLambda** function. The API Gateway responds with the result from the Lambda function to the user.

In the above image you can see the fiddler requests and responses on line number 7 and 11 for the 307 redirect and a request/response from the API Gateway.


##### Second request
 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/test-image-resize-2nd-req.png  "test image resize 2nd request"){:width="80%"}

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/fiddler-image-resize-2nd-req.png  "fiddler image resize 2nd request"){:width="80%"}

When the next request comes to the CloudFront for the same resized image, CF makes a request to the origin server and this time the image is created and found, it responds with to the client and caches it in the CF Distribution, as you can see in the fiddler snapshot on line number 18; also note that it's a miss from the cloudfront.

##### All other subsequest requests
 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/test-image-resize-3rd-req.png  "test image resize 3rd request"){:width="80%"}

 ![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/fiddler-image-resize-3rd-req.png  "fiddler image resize 3rd request"){:width="80%"}


All subsequent requests for the same resized image will be served from CloudFront until the cache expires at which point CloudFront gets a fresh copy of the image from Origin server. As you can see from the response for the request on line number 26, it's a Hit from the cloudfront. Also note that in the first image, that the time it takes to load is 148 ms as compared to 241 ms where CF had to fetch it from the Origin server.

Of course, since we set up cache control, the same image will be served from the memory cache for the same browser until it expires.


#### Summary

In this post we saw how we can accomplish on demand image resizing using AWS Lambda, Amazon S3 and Amazon API Gateway and serving through Amazon CloudFront CDN. 

By serving appropriately sized images, we saw how we can save many bytes of data and improve the performance of a webpage. 

Though there’s a learning curve understanding AWS services and figuring out how various services work together, it’s worth experimenting and investing some time to see if it works for you as it’s got a lot of great features and functionalities. 

The flow we saw here is one of the many options you can accomplish the same result by using various other combinations of AWS services. AWS offers a pay-as-you-go approach for pricing, which means you pay only for the individual services you need, for as long as you use them; so make sure that you choose right services that meet your needs.

