You probably already run Nightmare on your dev machine or on some kind of build or test server (like AWS EC2). 
But now, you are reading this guide because you have decide to avoid all the hassle of maintaining servers
and run Nightmare on AWS Lambda serverlessly.

# Quick Start


## Install dependencies


> WIP! In order to complete this tutorial, we need to publish `nightmare-lambda-pack` npm package and 
> get our PR merged into `xvfb` package creator. (or create fork). After those steps completed
> we would be able to use the following installation line: `npm install nightmare nightmare-lambda-pack xvfb`.
> However for the moment we have to pull dev versions of those files via wget.

```
cd my-lambda-tutorial
npm init -y
npm install nightmare 
mkdir -p lib/bootstrap
wget -O lib/bootstrap/nightmare-lambda-pack.js https://raw.githubusercontent.com/dimkir/nightmare-lambda-tutorial/master/lib/bootstrap/nightmare-lambda-pack.js
wget -O lib/bootstrap/xvfb.js https://raw.githubusercontent.com/dimkir/nightmare-lambda-tutorial/master/lib/bootstrap/xvfb.js


```


## Write your lambda function source code

When writing your lambda logic, you will have to add binary pack installation line outside of the event handler
and wrap actual application logic within  callback to `xvfb.start()`  (and remember to call `xvfb.stop()`). 

After you've completed those steps your source would look like this:

```js

var 
  binaryPack = require('./lib/bootstrap/nightmare-lambda-pack'), // WIP: should be `require('nightmare-lambda-pack')`
  Xvfb       = require('./lib/bootstrap/xvfb'),                  // WIP: should be `require('xvfb')`
  Nightmare  = require('nightmare')
;

var isOnLambda = binaryPack.isRunningOnLambdaEnvironment;

var electronPath = binaryPack.installNightmareOnLambdaEnvironment();

exports.handler = function(event, context){
    var xvfb = new Xvfb({
        xvfb_executable: '/tmp/pck/Xvfb',  // Xvfb executable will be at this path when unpacked from nigthmare-lambda-pack
        dry_run: !isOnLambda         // in local environment execute callback of .start() without actual execution of Xvfb (for running in dev environment)
    });
    
    xvfb.start((err, xvfbProcess) => {

        if (err) context.done(err);

        function done(err, result){
            xvfb.stop((err) => context.done(err, result));
        }

        // ...
        // Main logic with call to done() upon completion or error
        // ...

        var nightmare = Nightmare({
          show: true,                   // show actual browser window as Nightmare clicks through
          electronPath: electronPath    // you MUST specify electron path which you receive from installation
        });

        nightmare
            .goto('https://duckduckgo.com')
            .type('#search_form_input_homepage', 'github nightmare')
            .click('#search_button_homepage')
            .wait('#zero_click_wrapper .c-info__title a')
            .evaluate(function () {
                return document.querySelector('#zero_click_wrapper .c-info__title a').href;
            })
            .end()
            .then(function (result) {
                console.log(result);
                done(null, result);  // done() instead of context.done()
            })
            .catch(function (error) {
                console.error('Search failed:', error);
                done(error);         // done() instead of context.done()
            });    

    });
};

```




## Create lambda function zip-package

Let's create deployment package:

```
zip -r nightmare-tutorial-function.zip index.js lib node_modules -x '*electron/dist*' 
ls -lh *.zip
```

The resulting zip package should be 3-4Mb in size.

Not that we add `-x '*electron/dist*' option to exclude 
`node_modules/nightmare/node_modules/electron/dist` 
subfolder which has heavy electron binaries, which are of no use when pushed to AWS Lambda environment.



>Note that `'*electron/dist*'` pattern will exclude any filepath (eg. `selectron/distrib`), so 
> if any of your direct or transitive dependency names happen to collide with this pattern, simply use 
> longer version of the exclusion pattern:
>```
>zip -r nightmare-tutorial-function.zip index.js lib node_modules \
>   -x '*node_modules/nightmare/node_modules/electron/dist*'
>```
>
>


## Create your Lambda function on AWS 

Now as you have zip-file with your lambda source, you are ready to 
create lambda function on AWS. 

We are going to create function with the following parameters:

  - Runtime `node4.3`
  - Memory size `1GB` (Selecting `1GB` of memory would automatically match up with faster instance which will unzip binary package in a jiffy)
  - Timeout of `60` seconds (Downloading & installing Electron along with Xvfb and running Nightmare crawl of the site will take longer than default timeout of 3 seconds. We recommend 60 or at least 30 seconds).
  - Deployment package zip from the previous step


There are  many ways you can create lambda function on AWS
  - AWS web console (you can do it yourself)
  - AWS CLI
  - Cloud Formation

In order to help you get up and running with your lambda function quickly, we have prepared a simple standalone helper script
`lambda-install-aws.sh`, which you can put into your projects directory.

```
wget -o lambda-install-aws.sh https://raw.githubusercontent.com/dimkir/nightmare-lambda-tutorial/master/bin/install/lambda-install-aws.sh
```

Now edit this script "Settings" section and set up your unique function name and region.

```
#############
#  Settings
#############

FUNCTION=nightmare-tut-hello
REGION=eu-west-1
DEPLOYMENT_PACKAGE_ZIP=nightmare-tutorial.zip

```

In order to run this script you need AWS CLI installed and credentials configured.

> Keep in mind that AWS permissions for initial creation of Lambda function should be relatively broad. 
> Because creation of lambda function requires usage of AWS IAM to create role and policy and later AWS Lambda to create function and configuration.
> If you use managed policies I would recommend Administrator as PowerUser won't be enough.


Now let's create our function on AWS:

```
./lambda-install-aws.sh
```

And you will get output like this:

```
[my-nightmare-tutorial]2$ ./lambda-install-aws.sh 
>>> Creating execution role ...
    Execution role name: [nightmare-tut-hello-lambda-execution-role]
    Execution role arn: [arn:aws:iam::326625058526:role/nightmare-tut-hello-lambda-execution-role]
>>> Creating execution policy...
>>> Creating lambda function...
{
    "CodeSha256": "nulOwznG/Cm9OaNvglEJedqL9OuFfUNPOznhE4cZMFs=", 
    "FunctionName": "nightmare-tut-hello", 
    "CodeSize": 3658846, 
    "MemorySize": 1024, 
    "FunctionArn": "arn:aws:lambda:eu-west-1:326625058526:function:nightmare-tut-hello", 
    "Version": "$LATEST", 
    "Role": "arn:aws:iam::326625058526:role/nightmare-tut-hello-lambda-execution-role", 
    "Timeout": 60, 
    "LastModified": "2017-03-18T12:53:21.106+0000", 
    "Handler": "index.handler", 
    "Runtime": "nodejs4.3", 
    "Description": ""
}

```



## Invoke your lambda function

We can use AWS CLI to invoke lambda and will pass two parameters: `done.log` 
will hold the result of call to `context.done()` 
and as payload we will send empty event `{}`


```
[my-nightmare-tutorial]$ aws lambda invoke --function-name nightmare-tut-hello --payload {} done.log
{
    "StatusCode": 200
}

echo ; cat done.log ; echo

# if everything worked well, you will see similar url
"https://github.com/segmentio/nightmare"

```



## How did we make Nightmare work on Lambda - The Full Picture


Lambda is amazing, but this awesomeness comes at cost of few restrictions and limitations. 
Thus setting up Nightmare "the usual way"  via `npm install nightmare` and uploading it to Lambda *won't work*.
In this tutorial we outline the recipe to go around those Lambda limitations and give some technical insights which
will help you develop intuition for running Nightmare on Lambda.



## The vanilla approach which won't work on Lambda

You would typically start writing source for your lambda function on local development machine following those steps:
 
 1. Install Nightmare `npm install nightmare` 
 2. Create `index.js` (source below)
 3. Create dummy event file to test lambda `echo {} > event.json` 
 4. Run `lambda-local -l index.js  -e event.json --timeout 30`  

> To run lambda on local machine with ease, we will use  [lambda-local](https://www.npmjs.com/package/lambda-local)
> command line utility which you can install on your dev machine via  `npm install -g lambda-local`. 


```js

// index.js 
var Nightmare = require('nightmare');       
var nightmare = Nightmare({ show: true });

exports.handler = function(event, context){

    nightmare
    .goto('https://duckduckgo.com')
    .type('#search_form_input_homepage', 'github nightmare')
    .click('#search_button_homepage')
    .wait('#zero_click_wrapper .c-info__title a')
    .evaluate(function () {
        return document.querySelector('#zero_click_wrapper .c-info__title a').href;
    })
    .end()
    .then(function (result) {
        console.log(result);
        context.done(null, result); 
    })
    .catch(function (error) {
        console.error('Search failed:', error);
        context.done(error);
    });

}
  
```


## Problem with missing Electron dependencies
Although this approach seems trivial, there's a bit of magic involved. 
In particular `npm install` will _not only_ install JavaScript sources for the dependencies, 
but also binary dependencies. And because Nightmare relies heavily on [Electron](http://electron.atom.io) under the hood, 
the binary executable `electron` will be installed into your projects subfolder 
 `node_modules/nightmare/node_modules/electron/dist`.  The `dist` folder will have few more supporting binaries including localization files and libs.

On your typical desktop distribution of Linux everything is going to work well. On most server distributions 
you will hit missing shared libraries error, which you can solve by following 
[instructions on solving Common Execution Problems](https://github.com/segmentio/nightmare#common-execution-problems). 
However the solution provided requires root access, which you do **not** have on Lambda.


So Electron requires:
  - a ton of static libraries which are not installed on Lambda (eg. libgtk+)
  - a real display or fake display framebuffer (eg. `Xvfb` ) to run successfully

In order to solve the problem we have [picked and compiled missing libraries](https://gist.github.com/dimkir/f4afde77366ff041b66d2252b45a13db) manually 
and now we need to upload to Lambda somehow.

## Lambda has limit to zip-package size
When we first bundled all the missing libraries into a zip-file to upload to Lambda, it turned out `58 Mb` 
whilst [Lambda has limit](http://docs.aws.amazon.com/lambda/latest/dg/limits.html) of `50 Mb` maximum for function zip-file.


The solution was simple - to package all the required binaries into a zip file and make it available on S3 in the region where your
function runs. Your lambda function should download the binary package to `/tmp` directory before each execution and unzip it.
 
This is why in the head of your lambda function you must use: 

```js
var binaryPack = require('./lib/bootstrap/nightmare-lambda-pack');

// returns path to electron executable eg. /tmp/pck/electron
var electronPath = binaryPack.installNightmareOnLambdaEnvironment();

exports.handler = function(event, context){
    ...
}

```

The `installNightmareOnLambdaEnvironment()` call would take care of pulling binary package from S3 into 
current running Lambda container before execution of your event handler. Also as the name suggests on your local
machine it will do nothing. So you can run lambda function locally with `lambda-local` or other runner.


If you want to inspect the binary Nightmare package (which includes Electron and Xvfb) for your region you can find some of them here:

 - Hosted on `eu-west-1` [nightmare-lambda-pck-with-xvfb-20170313-1726-43.zip](http://static.apptentic.com/tools/electron/nightmare-lambda-pck-with-xvfb-20170313-1726-43.zip) (58 Mb)
 - Hosted on `us-west-1` TBD


## Adding virtual display framebuffer to Lambda (Xvfb)

To run display application on headless systems, you would run virtual framebuffer (`Xvfb`) as a background daemon or via runner `xvfb-run.sh`. 
In both cases `Xvfb` is run _before_ we run our application. In order to achieve the same result and run `Xvfb` before Nightmare/Electron in our Lambda function,
we will use [xvfb package](https://www.npmjs.com/package/xvfb). This package will also take care of closing cleanly `Xvfb` after we have finished Nightmare execution.



The example below wraps your main logic with calls to start and stop Xvfb 

> Note: in this example `Xvfb` will be forced to run regardless of whether you already got a real display or not (eg. on your dev machine).
> Instructions on how to run `Xvfb` only when on AWS Lambda environment are further below.

```js
var Xvfb = require('./lib/bootstrap/xvfb');

// ...

exports.handler = function(event, context){


var xvfb = new Xvfb({ xvfb_executable: '/tmp/pck/Xvfb' }); // this is location of Xvfb executable, after binary pack unzipped

 xvfb.start((err, xvfbProcess) => {
     
     if (err) context.done(err);

     function done(err, result){
        xvfb.stop((err) => context.done(err, result)); 
     }

     // ... 
     // Here goes the logic of your actual lambda function
     // note that you must call done() instead of context.done() to close Xvfb correctly.
     // ...


 });

}

// ...


```


## Code sample for final solution 

Now we know that in order to run Nightmare successfully as lambda function we need to:
 1. Download the binary lambda pack
 2. Wrap our main logic in calls to `Xvfb` 



```js
var 
  binaryPack = require('./lib/bootstrap/nightmare-lambda-pack'),
  Xvfb       = require('./lib/bootstrap/xvfb')
;

var isOnLambda = binaryPack.isRunningOnLambdaEnvironment;

var electronPath = binaryPack.installNightmareOnLambdaEnvironment();

exports.handler = function(event, context){
    var xvfb = new Xvfb({
        xvfb_executable: '/tmp/pck/Xvfb',  // Xvfb executable will be at this path when unpacked from nigthmare-lambda-pack
        dry_run: !isOnLambda         // in local environment execute callback of .start() without actual execution of Xvfb (for running in dev environment)
    });
    
    xvfb.start((err, xvfbProcess) => {

        if (err) context.done(err);

        function done(err, result){
            xvfb.stop((err) =>{
                context.done(err, result);
            });
        }

        // ...
        // Main logic with call to done() upon completion or error
        // ...

    });
};
```

Now as you know how the solution works, see below simple instructions to make it work:








