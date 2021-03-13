---
title: "End to end guide for developing dockerized lambda in Nodejs (Typescript), Terraform and SAM cli"
date: "2021-03-13T09:00:00.009Z"
category: "development"
---

This is a full guide to locally develop and deploy a backend app with [a recently released container image feature for lambda on AWS](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-container-image-support/). 

Needless to say, if you are a great fan of Docker, you would know how amazing it is. What you test on local is what you get when you deploy it, at least at the container level.

Since this feature is quite new, there has been **lots of rabbit holes** I fell into, and I'm sure others will too, so I will break every piece of rabbit hole down so that nobody gets trapped into it.

# Reason to use Terraform and SAM CLI together

Well, it seems that Terraform supports building a Docker image and deploying it to ECR out of the box, but after lots of digging, I noticed that things would get simpler if I just build docker image in another pipeline and deploy it with a few lines of shell script. So Terraform will used to define resources excluding the build and deployment process. There's no problem with that.

And, what SAM CLI? Terraform cannot replace SAM CLI and vice versa. SAM cli is useful in developing local lambdas because it automatically configures endpoints for each lambda and greatly removes barriers to the initial setup. Since lambda functions are 'special' in the way that they only get 'booted up and called' when they are invoked (unlike EC2 or Fargate), just writing a plain `.ts` file and `ts-node my-lambda.ts` would not make it work. Of course there are many other solutions to this matter (such as `sls`) but in this guide I will just use SAM CLI. But for many reasons SAM makes me want to use other better solutions if any... The reason follows right below.

_Disclaimer for the people who are looking for how to 'hot-reload' Dockerfile for typescript or javascript based lambda_: it won't work smoothly as of now. The best bet is to use `nodemon` to watch a certain directory to trigger `sam build` every single time, and in another shell launch `sam local start-api`. It works as expected, but the current problem I see from here is that every single time it `sam build`s, it would make another Docker image and another and so on, so there will be many useless dangling images stacked up in your drive, which you will need to delete manually because SAM CLI does not support passing in a argument that's equivalent to `docker run --rm`. Anyways that's the story, so this is the reason I might want to try some other solutions. [More on this on the relevant issue on Github](https://github.com/aws/aws-sam-cli/issues/921).

Ok. Now let's write some code.

# Setup AWS for Terraform

First, make sure that you've installed and authorized on your AWS CLI. Installing AWS CLI is kind of out of scope here, so [please follow the guide on AWS](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html).

After successful installation, run:

```bash
aws configure
```

You will be prompted to enter Access Key ID and Secret Access Key. Depending on your situation, there are different ways of how you can handle this, but for the sake of simplicity we can make one user (from AWS console. You will probably only use it for 
'Programmatic access') that would have these policies for applying Terraform codes.

![./add-user.png](./add-user.png)

This one for setting S3 bucket as a backend:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::tf-backend"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::tf-backend/path/to/my/key"
    }
  ]
}
```

And this one for locking the state:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/tf-state-locks"
    }
  ]
}
```

And the next one is quite tricky; because we will temporarily enable permissions related to managing IAM because we will first need to make a role from which we could `assumeRole` whenever we try to plan and apply our IaC.

For now we can just go onto AWS console and make this policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "iam:*",
                "sts:*"
            ],
            "Resource": "*"
        }
    ]
}
```

Make sure you will need to narrow down to specific actions and resources used after everything is done.

![create-policy](./create-policy.png)

Now, now that you've made three distinct policies (or all in one, depending on your preferences), attach them to the user that you've just crated for running `aws configure`.

# Setup Terraform

If you haven't already, [install terraform by following an instruction from the official website](https://learn.hashicorp.com/tutorials/terraform/install-cli). Just download the binary and move it to the `bin` folder.

Now verify version of terraform

```bash
➜  test terraform --version
Terraform v0.14.5

Your version of Terraform is out of date! The latest version
is 0.14.8. You can update by downloading from https://www.terraform.io/downloads.html
```

And then make `main.tf` file in your project directory (I personally put it into `IaC` folder because there will another folder for the 'real' `.ts` codes for the backend):

## `main.tf`

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
  }
}

provider "aws" {
  profile = "default"
  region  = "us-west-2" # you will need to change this to your region
}
```

Now, run `terraform init`:

```bash
➜  test terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 3.27"...
- Installing hashicorp/aws v3.32.0...
- Installed hashicorp/aws v3.32.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Then, we will need to add s3 backend and state locking. But before then, make a table on Dynamodb and also a bucket on S3, each for hosting IaC backend and locking the state.

Now we will need to add more to the policy on DynamoDB we created because we want to create a table:

```bash
  "dynamodb:CreateTable",
  "dynamodb:DescribeTable",
  "dynamodb:Scan",
  "dynamodb:Query",
```

Then you could write this code (by the way, it may be a good idea to put this below IaC in a different general-purpose repository because the current repository is meant to be only used for lambda-related resouces. But for the sake of this article I will just write it away here):

```bash
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "tf-state-locks"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

And then, also add S3 backend (you will need to add relevant IAM policies too here, but since we know how to do it, I will cut the explanation):

```bash
resource "aws_s3_bucket" "b" {
  bucket = "tf-backend"
  acl    = "private"

  versioning {
    enabled = true
  }
}
```

Now, run `terraform apply`, verify the changes, and enter `yes`. DynamoDB table and S3 Bucket should have been created. Here's the code so far:

## `main.tf`

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
  }
}

provider "aws" {
  profile = "default"
  region  = "us-west-2" # you will need to change this to your region
}

+ resource "aws_dynamodb_table" "terraform_state_lock" {
+   name           = "tf-state-locks"
+   read_capacity  = 5
+   write_capacity = 5
+   hash_key       = "LockID"
+   attribute {
+     name = "LockID"
+     type = "S"
+   }
+ }

+  resource "aws_s3_bucket" "terraform_backend" {
+    bucket = "tf-backend"
+    acl    = "private"

+    versioning {
+      enabled = true
+   }
+  }
```

Now, add s3 backend and state lock:

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
  }
+ backend "s3" {
+   profile = "localtf"
+   bucket  = "my-iac" # change the bucket name to yours
+   key            = "your-stack-name"
+   region         = "us-west-2" # change to your region
+   dynamodb_table = "terraform-lock"
+ }
}

provider "aws" {
  profile = "default"
  region  = "us-west-2" # you will need to change this to your region
}

resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "tf-state-locks"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

 resource "aws_s3_bucket" "terraform_backend" {
   bucket = "tf-backend"
   acl    = "private"

   versioning {
     enabled = true
  }
 }
```

We are also going to use Docker provider, so add that too:

## `main.tf`
```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
 +  docker = {
 +    source  = "kreuzwerker/docker"
 +    version = ">= 2.8.0"
 +  }
  }
  backend "s3" {
    profile = "localtf"
    bucket  = "my-iac" # change the bucket name to yours
    key            = "your-stack-name"
    region         = "us-west-2" # change to your region
    dynamodb_table = "terraform-lock"
  }
}


provider "aws" {
  profile = "default"
  region  = "us-west-2" # you will need to change this to your region
}

resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "tf-state-locks"
  read_capacity  = 5
  write_capacity = 5
  hash_key       = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

 resource "aws_s3_bucket" "terraform_backend" {
   bucket = "tf-backend"
   acl    = "private"

   versioning {
     enabled = true
  }
 }
```

Now because you've added a backend and another provider, we will need to run `terraform init` again, and then `terraform apply`. Run it.

# Setting up lambda

Now we will need to develop lambda on the local machine. Install SAM CLI:

```bash
brew tap aws/tap

brew install aws-sam-cli
```

Note that the outdated versions would not support running Docker containers, so make sure that your version is the latest.

```bash
test:(dev) sam --version
SAM CLI, version 1.20.0
```

Now, we won't run `sam --init`, because it will make it difficult to make the server into a monorepo structure. The reason that we will want to make it into a monorepo is that it will make it much easier to propery dockerize every single lambda and deploy it with dependencies that each of them only require to have. Instead we will use _lerna_ to initialize the server folder.

```bash
mkdir server
```

```bash
|
- IaC
- server
```

And as usual:

```bash
cd server

npm i -g lerna

lerna init
```

Then it will give you this layout:

```bash
➜  test git:(master) ✗ tree
.
├── IaC
│   └── main.tf
└── server
    ├── lerna.json
    ├── package.json
    └── packages
```

Then, add your first function package. For the sake of this example, let's assume that we want to make a REST API, composed of many lambdas, each returning 'hello' as a response in different languages (which is totally useless in reality, but at least useful here). Our first lambda will be an English one.

```bash
➜  server git:(master) ✗ lerna create hello
lerna notice cli v3.18.1
lerna WARN ENOREMOTE No git remote found, skipping repository property
package name: (hello) ls
version: (0.0.0)
description:
keywords:
homepage:
license: (ISC)
entry point: (lib/hello.js)
git repository:
About to write to /Users/jm/Desktop/test/test/server/packages/hello/package.json:

{
  "name": "ls",
  "version": "0.0.0",
  "description": "> TODO: description",
  "author": "9oelM <hj923@hotmail.com>",
  "homepage": "",
  "license": "ISC",
  "main": "lib/hello.js",
  "directories": {
    "lib": "lib",
    "test": "__tests__"
  },
  "files": [
    "lib"
  ],
  "scripts": {
    "test": "echo \"Error: run tests from root\" && exit 1"
  }
}


Is this OK? (yes)
lerna success create New package ls created at ./packages/hello
```

Now the directory structure will look like this:

```bash
➜  test git:(master) ✗ tree -I node_modules
.
├── IaC
│   └── main.tf
└── server
    ├── lerna.json
    ├── package.json
    └── packages
        └── hello
            ├── README.md
            ├── __tests__
            │   └── hello.test.js
            ├── lib
            │   └── hello.js
            └── package.json

6 directories, 7 files
```

Now, under `server`, we will need to add some utils to build and invoke the function locally. Add modify the `server/package.json` as follows and of course, run `npm i` again:

```json
{
  "devDependencies": {
    "@types/node": "^14.14.32",
    "concurrently": "^6.0.0",
    "lerna": "^4.0.0",
    "nodemon": "^2.0.7",
  },
  "scripts": {
    "start": "concurrently \"npm run watch\" \"npm run api\"",
    "watch": "nodemon",
    "api": "sam local start-api",
    "docker:login": "aws ecr get-login-password --region {your-aws-region} | docker login --username AWS --password-stdin {your-aws-account}.dkr.ecr.{your-aws-region}.amazonaws.com"
  },
  "name": "server"
}
```

To add some explanatioon to what we are trying to do: these `devDependencies` are going to be package-wide dependencies. These are not specific to any one of the functions that we are going to build; They will help in tooling general stuffs. That's what we put them here.

Dependencies:

- `@types/node`: we will need this to give proper type definitions for 'built-in node' modules like `fs` or `path`.
- `concurrently`: just a script runner.
- `lerna`: you know it.
- `nodemon`: this will help us watch a directory and build Docker image again.

Scripts:

- `start`, `watch`, `api`: we will need these to launch our lambda function locally and invoke it.
- `docker:login`: we will need this to push the docker image to ECR for deployment. More on this later.

Now, you need to create `template.yml` for SAM cli to consume and run what we want to run.

```yml
########################################
# DO NOT USE THIS TEMPLATE TO DEPLOY TO AWS!!!
# ONLY USE IT FOR LOCAL TESTING
########################################

AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  hello api
  
# More info about Globals: https://github.com/awslabs/serverless-application-model/blob/master/docs/globals.rst
Globals:
  Function:
    Timeout: 3

Resources:
  helloRoomFunction:
    Type: AWS::Serverless::Function
    Metadata:
      DockerTag: nodejs14.x-v1
      DockerContext: ./packages/hello
      Dockerfile: Dockerfile
    Properties:
      # will automatically use local one if testing on local
      ImageUri: {your-aws-account-id}.dkr.ecr.{your-aws-region}.amazonaws.com/hello-in-eng
      # Note: If the PackageType property is set to Image, then either ImageUri is required, 
      # or you must or you must build your application with necessary Metadata entries in the AWS SAM template file. 
      # For more information, see Building applications.
      PackageType: Image
      FunctionName: HelloInEngFunction
      Events:
        helloRoom:
          Type: Api
          Properties:
            Path: /hello-in-eng
            Method: get
```

We won't be able to run `sam build` or `sam local start-api` yet, because we still need to setup `Dockerfile` and ECR repository.

So far we have added `template.yml` for running SAM CLI:

```bash
➜  server git:(master) ✗ tree -I node_modules
.
├── lerna.json
├── package.json
├── packages
│   └── hello
│       ├── README.md
│       ├── __tests__
│       │   └── hello.test.js
│       ├── lib
│       │   └── hello.js
│       └── package.json
└── template.yml
```

Now we will add `Dockerfile` in `packages/hello/`.

```bash
cd packages/hello

touch Dockerfile
```

This will be the content for Dockerfile:

```dockerfile
FROM amazon/aws-lambda-nodejs:14 AS builder
WORKDIR /usr/app

COPY package*.json tsconfig.json ./
RUN npm install

COPY ./lib ./lib/
RUN npm run build

RUN ls -la # for debugging

FROM amazon/aws-lambda-nodejs:14
# CMD ["path-to-file.function-name"]
WORKDIR /usr/app

COPY package*.json ./
RUN npm install --only=prod
COPY --from=builder /usr/app/lib /usr/app/lib
CMD [ "/usr/app/lib/index.handler" ]
```

To go through it line by line:

- `amazon/aws-lambda-nodejs:14` is the official amazon image for lambda. Current LTS of nodejs is 14, so we are using this. `AS builder` is related to multi-stage builds in Docker; it helps reduce the final Docker image size. Basically in this `builder` stage, we will only build the output to be included in the final image, and any dependencies installed in this step won't be included in the final output image.
- `WORKDIR /usr/app`: inside the docker image, set the working directory as `/usr/app`. There isn't any `app` folder in a normal docker image, so it will make `app` directory. We will put the compiled js code there.
- `COPY package*.json tsconfig.json ./`: we need these files for compiling typescript into javascriptt files.
- `npm install`: it will install dependencies.
- `npm run build`: it will compile typescript code into js.
- `RUN ls -la # for debugging`: it is merely for debugging. While building, docker will output what's inside there at that time, for you to verify if you are doing what you intended to do.
- `FROM amazon/aws-lambda-nodejs:14`: this is the second build stage in Docker. All outputs from the previous stage will be discarded in this stage unless explicitly specified to be included.
- `RUN npm install --only=prod`: it will only install `dependencies` but `devDepdencies`.
- `COPY --from=builder /usr/app/lib /usr/app/lib`: it explicitly refers to the previous `builder` stage to copy whatever that was inside `/usr/app/lib` to the current `/usr/app/lib`. In this case, it will copy all compiled javascript code.
- `CMD [ "/usr/app/lib/index.handler" ]`: the command should be `path-to-lambda-handler-without-extension.handler`. That's just how it works.

Now we've added a Dockerfile. Now let's setup basic environment for lambda:

```bash
cd packages/hello

➜  hello git:(master) ✗ tsc --init
message TS6071: Successfully created a tsconfig.json file.
```

You will need to modify `tsconfig` to use modern javascript features; Most prominently, add the following. This will allow you to use `Promise` API. I recommend turning other options too, especially those related to strict-type checking:

```json
{
  "compilerOptions": {
    /* Basic Options */
    // "incremental": true,                   /* Enable incremental compilation */
    "target": "es5",                          /* Specify ECMAScript target version: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017', 'ES2018', 'ES2019' or 'ESNEXT'. */
    "module": "commonjs",                     /* Specify module code generation: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', or 'ESNext'. */
-    "lib": [],                             /* Specify library files to be included in the compilation. */
+    "lib": ["ES2015"],                             /* Specify library files to be included in the compilation. */
    // "allowJs": true,                       /* Allow javascript files to be compiled. */
    // "checkJs": true,                       /* Report errors in .js files. */
    // "jsx": "preserve",                     /* Specify JSX code generation: 'preserve', 'react-native', or 'react'. */
    // "declaration": true,                   /* Generates corresponding '.d.ts' file. */
    // "declarationMap": true,                /* Generates a sourcemap for each corresponding '.d.ts' file. */
    // "sourceMap": true,                     /* Generates corresponding '.map' file. */
    // "outFile": "./",                       /* Concatenate and emit output to single file. */
    // "outDir": "./",                        /* Redirect output structure to the directory. */
    // "rootDir": "./",                       /* Specify the root directory of input files. Use to control the output directory structure with --outDir. */
    // "composite": true,                     /* Enable project compilation */
    // "tsBuildInfoFile": "./",               /* Specify file to store incremental compilation information */
    // "removeComments": true,                /* Do not emit comments to output. */
    // "noEmit": true,                        /* Do not emit outputs. */
    // "importHelpers": true,                 /* Import emit helpers from 'tslib'. */
-    // "downlevelIteration": true,            /* Provide full support for iterables in 'for-of', spread, and destructuring when targeting 'ES5' or 'ES3'. */
+    "downlevelIteration": true,            /* Provide full support for iterables in 'for-of', spread, and destructuring when targeting 'ES5' or 'ES3'. */
    // "isolatedModules": true,               /* Transpile each file as a separate module (similar to 'ts.transpileModule'). */

    /* Strict Type-Checking Options */
    "strict": true,                           /* Enable all strict type-checking options. */
    // "noImplicitAny": true,                 /* Raise error on expressions and declarations with an implied 'any' type. */
    // "strictNullChecks": true,              /* Enable strict null checks. */
    // "strictFunctionTypes": true,           /* Enable strict checking of function types. */
    // "strictBindCallApply": true,           /* Enable strict 'bind', 'call', and 'apply' methods on functions. */
    // "strictPropertyInitialization": true,  /* Enable strict checking of property initialization in classes. */
    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */

    /* Additional Checks */
    // "noUnusedLocals": true,                /* Report errors on unused locals. */
    // "noUnusedParameters": true,            /* Report errors on unused parameters. */
    // "noImplicitReturns": true,             /* Report error when not all code paths in function return a value. */
    // "noFallthroughCasesInSwitch": true,    /* Report errors for fallthrough cases in switch statement. */

    /* Module Resolution Options */
    // "moduleResolution": "node",            /* Specify module resolution strategy: 'node' (Node.js) or 'classic' (TypeScript pre-1.6). */
    // "baseUrl": "./",                       /* Base directory to resolve non-absolute module names. */
    // "paths": {},                           /* A series of entries which re-map imports to lookup locations relative to the 'baseUrl'. */
    // "rootDirs": [],                        /* List of root folders whose combined content represents the structure of the project at runtime. */
    // "typeRoots": [],                       /* List of folders to include type definitions from. */
    // "types": [],                           /* Type declaration files to be included in compilation. */
    // "allowSyntheticDefaultImports": true,  /* Allow default imports from modules with no default export. This does not affect code emit, just typechecking. */
    "esModuleInterop": true                   /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */
    // "preserveSymlinks": true,              /* Do not resolve the real path of symlinks. */
    // "allowUmdGlobalAccess": true,          /* Allow accessing UMD globals from modules. */

    /* Source Map Options */
    // "sourceRoot": "",                      /* Specify the location where debugger should locate TypeScript files instead of source locations. */
    // "mapRoot": "",                         /* Specify the location where debugger should locate map files instead of generated locations. */
    // "inlineSourceMap": true,               /* Emit a single file with source maps instead of having a separate file. */
    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */

    /* Experimental Options */
    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
  }
}
```

Modify `packages/hello/package.json` too. Be noted that any dependencies you add to be included in the final, compiled output code (javascript) will need to be added to `dependencies`, not `devDependencies`:

## `packages/hello/package.json`

```json
{
  "name": "hello",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "build": "tsc --project ./tsconfig.json"
  },
  "devDependencies": {
    "@types/aws-lambda": "^8.10.72",
    "typescript": "^4.2.3"
  }
}
```

Now, add a really simple lambda:

## `packages/hello/lib/index.ts`

```ts
import {
  APIGatewayProxyEvent,
  APIGatewayProxyResult
} from "aws-lambda";

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  return {
    statusCode: 200,
    body: `hello`
  }
}
```

So far, we have created these:
```bash
➜  server git:(master) ✗ tree -I node_modules
.
├── lerna.json
├── package.json
├── packages
│   └── hello
│       ├── Dockerfile
│       ├── README.md
│       ├── __tests__
│       │   └── hello.test.js
│       ├── lib
│       │   ├── hello.js
│       │   └── index.ts
│       ├── package.json
│       └── tsconfig.json
└── template.yml
```

Oh, and you can delete `__tests__` and `lib/hello.js` because we are not using them. Anyways, now we are kind of ready to build this function into a docker image. Let's try it:

```bash
➜  packages git:(master) ✗ cd hello
➜  hello git:(master) ✗ pwd
/Users/jm/Desktop/test/test/server/packages/hello
➜  hello git:(master) ✗ docker build -t hello .
[+] Building 2.3s (15/15) FINISHED
 => [internal] load build definition from Dockerfile                                                                                             0.0s
 => => transferring dockerfile: 445B                                                                                                             0.0s
 => [internal] load .dockerignore                                                                                                                0.0s
 => => transferring context: 2B                                                                                                                  0.0s
 => [internal] load metadata for docker.io/amazon/aws-lambda-nodejs:14                                                                           0.0s
 => [internal] load build context                                                                                                                0.0s
 => => transferring context: 438B                                                                                                                0.0s
 => [builder 1/7] FROM docker.io/amazon/aws-lambda-nodejs:14                                                                                     0.0s
 => CACHED [builder 2/7] WORKDIR /usr/app                                                                                                        0.0s
 => CACHED [stage-1 3/5] COPY package*.json ./                                                                                                   0.0s
 => CACHED [stage-1 4/5] RUN npm install --only=prod                                                                                             0.0s
 => CACHED [builder 3/7] COPY package*.json tsconfig.json ./                                                                                     0.0s
 => CACHED [builder 4/7] RUN npm install                                                                                                         0.0s
 => [builder 5/7] COPY ./lib ./lib/                                                                                                              0.0s
 => [builder 6/7] RUN npm run build                                                                                                              1.6s
 => [builder 7/7] RUN ls -la # for debugging                                                                                                     0.3s
 => [stage-1 5/5] COPY --from=builder /usr/app/lib /usr/app/lib                                                                                  0.1s
 => exporting to image                                                                                                                           0.1s
 => => exporting layers                                                                                                                          0.0s
 => => writing image sha256:f2d79403b039ee76e286564e93382d3e1268024fd053f2f8e0e8c6f5d73b1403                                                     0.0s
 => => naming to docker.io/library/hello                                                                                                         0.0s
```

Everything's cool, docker build succeeded. You can try running the image and test the request:

![docker first test](./docker-first-test.png)

OK. It shows the response we've created in `index.ts`. That's good. Now, let's setup 'hot reload' for the server. This is where SAM CLI should startt to come in. But before then, we will need to make a ECR repository with terraform. Let's go back to terraform for a while.

# Back to terraform: assume role and ECR

Now, we will need to create a role first because we will relay on that role to get required permissions to create whatever resource we want to. This is called 'assuming a role', and the reason why it's deemed to be a good practice is that you won't have to create multiple credentials (probably multiple users) to do certain thing that requires permissions. Instead, you _borrow_ the permission for the period of time when you plan and apply the changes in the resources.

So how do we do it? First, let's create `hello_role.tf`:

## `hello_role.tf`

```bash
resource "aws_iam_role" "hello" {
  name = "hello_role"
  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Principal" : {
          # get 'localtf' user's ARN from AWS IAM console
          # it should look like: arn:aws:iam::{aws-account-id}:user/localtf
          # example: arn:aws:iam::123456789:user/localtf
          "AWS" : "arn:aws:iam::123456789:user/localtf"
          "Service" : [
            "lambda.amazonaws.com"
          ]
        },
        "Action" : "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_policy" "hello" {
  name        = "hello_policy"
  description = "policy needed to run hello server stack"

  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Action" : [
          "lambda:*",
          "iam:*",
          "ecr:*",
          "cloudformation:*",
          "apigateway:*",
          "logs:*",
          "route53:*",
          "acm:*",
          "cloudfront:*",
          "ec2:*"
        ],
        "Effect" : "Allow",
        "Resource" : "*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "hello" {
  role       = aws_iam_role.hello.name
  policy_arn = aws_iam_policy.hello.arn
}
```

For the sake of this article, we won't be diving deep into specific policies, so we will just allow almost all resources without specifying them in detail. For real-world usage, you will have to define exact statements giving just the right permissions.

What we are doing here, essentially, is that we are allowing `localtf` user to assume the role of `hello_role` that possesses all policies to run the hello server stack. This is called 'creating a trust relationship' (you will see this if you do this process on AWS). This way, `localtf` won't always have to hold all permissions it needs. It only acquires them only when needed (i.e. deploying)

Once you are done writing `hello_role.tf`, run `terraform apply` to make changes.

Now, go back to `main.tf` and add:

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
   docker = {
     source  = "kreuzwerker/docker"
     version = ">= 2.8.0"
   }
  }
  backend "s3" {
    profile = "localtf"
    bucket  = "my-iac" # change the bucket name to yours
    key            = "your-stack-name"
    region         = "us-west-2" # change to your region
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  profile = "default"
  region  = "us-west-2" # you will need to change this to your region
+ assume_role {
+   role_arn     = "arn:aws:iam::{your-account-id}:role/hello_role"
+   session_name = "terraform"
+ }
}

# you can create this resource in other repository because it's not specific to this project
# resource "aws_dynamodb_table" "terraform_state_lock" {
#   name           = "tf-state-locks"
#   read_capacity  = 5
#   write_capacity = 5
#   hash_key       = "LockID"
#   attribute {
#     name = "LockID"
#     type = "S"
#   }
# }

# you can create this resource in other repository because it's not specific to this project
# resource "aws_s3_bucket" "terraform_backend" {
#   bucket = "tf-backend"
#   acl    = "private"

#   versioning {
#     enabled = true
#   }
# }
```

Once you add `assume_role`, now you can create any resources you want, using the permissions given by that role. Let's now make an ECR repository. Make `ecr.tf`: