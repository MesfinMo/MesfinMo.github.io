---
layout: post
date: 2018-03-31T20:25:30Z
title: "Uploading a specific folder of your project from your repository to amazon s3 bucket using Travis CI"
description: A developer pushes code to a github repository, Travis CI gets notified, it then starts building the code, run tests and commands based on scripts specified in ‘.travis.yml’ file. In our case, one of the commands is to deploy assets folder to amazon s3 bucket after a successful build. In this post we’ll step-by-step look into how we can use Travis CI to deploy a site ‘assets’ folder of our project to amazon s3 bucket automatically which is triggered by a code push to a Github repository.

active: true
mainFeatured: true
image: https://d3u9etne7s6j1g.cloudfront.net/blog/img/600x600/github-travis-amazons3.png
cropImage: https://d3u9etne7s6j1g.cloudfront.net/blog/img/600x200/github-travis-amazons3.png
img-credit-url: 
img-credit-desc: 
tags:
  - travis CI
  - github
  - Amazon s3 bucket
author-avatar: https://d3u9etne7s6j1g.cloudfront.net/base/img/60x60/author-avatar-default.jpg
author-name: Moses
author-title: 
---

I recently came across the need to serve a website’s assets(e.g. JavaScript, CSS and images) from a content delivery network (CDN). Using CDN to serve up assets not only helps accelerate webpage load time, by caching them on to servers nearest to visitors, but also helps in SEO, as website speed is one of google ranking factors.

After deciding that I would use amazon CloudFront, I was looking for an automated way of deploying these assets to amazon s3 bucket, which CloudFront uses as an origin server, instead of a manual approach whereby we sign in to amazon s3 console and use the upload feature everytime we make changes.

In this post we’ll step-by-step look into how we can use Travis CI to deploy a site ‘assets’ folder of our project to amazon s3 bucket automatically which is triggered by a code push to a Github repository.

#### Prerequisites
- Have an amazon aws account
- Have a GitHub account and install git on your local computer. I assume that you have a basic knowledge of GitHub and git.
- Install jekyll to create and run static sites.
- Have Visual Studio Code for code editing.

Looking at the diagram above, a developer pushes code to a github repository, Travis CI gets notified, it then starts building the code, run tests and commands based on scripts specified in ‘.travis.yml’ file. In our case, one of the commands is to deploy assets folder to amazon s3 bucket after a successful build. We’ll see this in more detail later in this post, but first let’s create a project.

#### Step 1 - Create a jekyll project

Jekyll is a simple but powerful static site generator which lets you create a site in no time by just running few commands; it is also deeply integrated with Github which offers a lot of built-in support. If you’re interested in learning more, see [jekyll documentation](https://jekyllrb.com/docs/home/){:target="_blank"}.

Now open up Visual Studio Code and go to the Integrated Terminal by pressing Ctrl+` and run the following commands:

<div class="cmd">
<pre class="prettyprint">
$ <span class="c-str">gem install jekyll bundler</span>
$ <span class="c-str">jekyll new siteAssets</span>
$ <span class="c-str">cd siteAssets</span>
$ <span class="c-str">bundle exec jekyll serve</span>
</pre>
</div>
Now browse to http://localhost:4000

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/jekyll-default-site.png  "jekyll default site"){:width="100%"}


#### Step 2 - Create amazon s3 bucket and IAM user

In this step, we’ll be creating a bucket where we’ll be uploading our site assets to, also while we’re here at aws management console, we’ll be creating IAM user with specific access permission to this newly created bucket so that we can give Travis CI grant to make API calls to aws to access this bucket on behalf of this user, which we’ll see later.

Sign in to [AWS Management Console](https://aws.amazon.com/console/){:target="_blank"} :

##### Create amazon s3 bucket

Open the amazon s3 console and click on ‘Create bucket’ button; on the popup window, enter bucket name and click Next through without changing the default settings.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/Amazon-s3-bucket.png  "Amazon s3 bucket"){:width="80%"}

##### Create IAM user

One of IAM best practices is to create individual IAM Users instead of using aws account root user credentials. When we create an IAM user, we can set permissions by attaching either AWS Managed Policies or Customer Managed Policies or both.

In this example we’ll be creating a Customer Managed Policy and then attach it to the user when we create it.

Since we’re creating this policy for Travis CI to be able to access the S3 bucket we just created, we make sure we include enough permission to do its job.

Open the IAM console and click on ‘Policies’ tab on the left menu and then click on ‘Create policy’ button and in the following page ‘Visual Editor’ tab, make the following selection.

- ‘Service’, select S3
- ‘Actions’ : ‘List’, select ‘List Bucket’
- ‘Actions’ : ‘Read’, select ‘GetObject’ and ‘GetObjectAcl’
- ‘Actions’ : ‘Write’, select ‘PutObject’ and ‘DeleteObject’
- ‘Actions’ : ‘Permissions management’, select ‘PutObjectAcl’
- ‘Resources’, select specific and then click on the ‘Add ARN’ link corresponding to ‘bucket’; on the popup window enter the ARN for the bucket we created in the previous step which is ‘arn:aws:s3:::blog-samples-site-assets’
- Click on ‘Add ARN’ link corresponding to ‘object’; on the popup window, enter the ARN followed by a forward slash and ‘\*’, as  ‘arn:aws:s3:::blog-samples-site-assets/\*’

> **Note**: You can find the ARN for a bucket by highlighting in from the list on S3 console and from the information window, click on ‘Copy Bucket ARN’

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/copy-bucket-ARN.png  "copy bucket ARN"){:width="80%"}

> **Note**: ‘blog-samples-site-assets’ is the name I gave to the bucket; it will be another one if you name yours differently.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/aws-create-policy.png  "aws create policy"){:width="80%"}

Click on ‘Review policy’ button on the bottom of the screen; give your new policy a meaningful name and description and finally click on the ‘Create policy’ button.

Now, click on ‘Users’ tab on the left menu; and then 

- click on the ‘Add user’ button; 
- on the next screen, give the new user a meaningful name on the ‘User name’ section and select ‘Programmatic access’ for ‘Access type’
- Click on the ‘Next: Permissions’ button
- On the next screen, click on ‘Attach existing policies directly’ button

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/attach-policy-to-user.png  "attach policy to user"){:width="80%"}

- Search the newly created policy by name
- Click on the check box on the left of the policy from the grid	 to select it
- Click on the ‘Next:Review’ button
- On the next page, click on the ‘Create user’ button

On the next page, you should be seeing a confirmation message that the new user is created; 

From the grid below, copy and paste the ‘Access key Id’ and ‘Secret access key’ values somewhere because we need them to configure travis CI later.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/iam-user-access-secret-key.png  "iam user access secret key"){:width="80%"}

> **Note**: this is the only place you can copy ‘Secret access key’ value, once you close out of this page you won’t be able to retrieve this information.

#### Step 3 - Create a Github repository and Push code to a repository

In this step, you create a github repository and push the code for the project you created in Step 1. Once you completed creating a repository: 

- open Terminal 
- change the current working directory to the project directory you created
- run the following commands:

<div class="cmd">
<pre class="prettyprint">
$ <span class="c-str">git init</span>
$ <span class="c-str">git add .</span>
$ <span class="c-str">git commit -m "First commit"</span>
$ <span class="c-str">git remote add origin remote git@github.com:MesfinMo/BlogSamples-siteAssets.git</span>
$ <span class="c-str">git push -u origin master</span>
</pre>
</div>
> **Note** : ‘git@github.com:MesfinMo/BlogSamples-siteAssets.git’ on the 4th command is the SSH url for the repository I created; rename it with the one you created.

#### Step 4 - Enable your project repository in Travis CI

Now go to travis-ci.org and sign in with your GitHub account.

Once you’re signed in, in the upper right corner, click on your name and choose ‘Profile’; you will see Travis CI has synchronized your GitHub repositories, including the one you just created in the previous step. Now, click on the grey button corresponding to the repository you want to enable.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/travis-enable-repo.png  "travis enable repo"){:width="80%"}

Once you see it’s enabled by the color change, click on the settings button next to it; and on the following page, on the ‘Environment Variables’ section, add values for ‘AWS_ACCESS_KEY_ID’ and ‘AWS_SECRET_ACCESS_KEY’ from what you’ve saved, by copying and pasting them, in the last part of ‘Step 2’.

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/travis-env-variables.png  "ttravis env. variables"){:width="80%"}

> **Note**: Make sure you add the value for ‘AWS_DEFAULT_REGION’ as well. 

#### Step 5 - Add .travis.yml file and ‘assets’ folder to the project

In this step, we’ll be updating our project by adding ‘.travis.yml’ file to the root as well as ‘assets’ folder where we’ll be saving our JavaScript, CSS and images into. 

##### Add ‘.travis.yml’ file

.travis.yml is a configuration file which is used by Travis CI to learn about your project and what jobs you want it to run.

In the root of your project add a new file and name it as ‘.travis.yml’ and then add the following code to it.

<div class="yml">
<pre class="prettyprint linenums">
language:<span class="c-str"> python</span>
python:
  - "2.7"
install: 
  - <span class="c-str">bundle install</span>
  - <span class="c-str">pip install awscli</span>
script:
  - <span class="c-str">bundle exec jekyll build</span>
after_success:
  - <span class="c-str">aws s3 sync --delete ./_site/assets  s3://blog-samples-site-assets --cache-control max-age=604800</span>
branches:
  only: 
    - <span class="c-str">master</span>
</pre>
</div>

There are basically two steps when travis CI performs a build :

1. install: install any dependencies required
2. script: run the build script

Travis CI also provides options to include custom commands if you want to run before the installation step ‘before_install’ as well as before ‘before_script’ or after ‘after_script’ the script step.

> **Note** : looking at the .travis.yml code above, there are no custom commands are required.

Travis CI also allows you to perform additional steps when your build succeeds or fails with ‘after_success’ or ‘after_failure’ options respectively. 
Read the [Customizing the Build](https://docs.travis-ci.com/user/customizing-the-build/){:target="_blank"} for an in-depth explanation.
sys

> **Note:**
>
> 1. We make use of the ‘after_success’ option to run aws cli command to sync our assets folder to s3 bucket.
> 2. The path to the ‘assets’ folder is ‘./_site/assets’ which doesn’t exist in our repository because ‘_site’ is excluded by the ‘.gitignore’ entry, but jekyll creates it after a successful build.


##### Add ‘assets’ folder

Now, you add a folder under the root and name it as ‘assets’; also add some images, css and javascript files as shown below: 

![alt text]({{ site.assetBaseUrl }}/blog/img/{{site.imgSize600}}/travis-yml-and-assets-folder.png  "travis yml and assets folder"){:width="80%"}

Once you complete that open Terminal and run the following commands:

<div class="cmd">
<pre class="prettyprint">
$ <span class="c-str">git add .</span>
$ <span class="c-str">git commit -m "add .travis.yml and assets folder"</span>
$ <span class="c-str">git push -u origin dev</span>
</pre>
</div>

Now the files you added in ‘assets’ folder should be deployed into the amazon s3 bucket you created, as shown in action below.

![alt text]({{ site.assetBaseUrl }}/blog/img/github-travis-amazon-s3.gif  "turbotax self employed"){:width="100%"}