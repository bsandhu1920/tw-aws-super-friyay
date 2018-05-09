# Create ECR repository and push image to ECR

## Configure AWS credentials
```
$ aws configure
AWS Access Key ID [None]: <AWS_ACCESS_KEY_ID>
AWS Secret Access Key [None]: <AWS_SECRET_ACCESS_KEY>
Default region name [None]: ap-southeast-2
Default output format [None]: json
```

## Deploy ECR repository via cloudformation

1. Create ecr.yml

2. Create cloudformation stack

    ```
    $ aws cloudformation create-stack --stack-name teststack --template-body "file://$(pwd)/ecr.yml"
    ```

    `--template-body` for template files
    `--template-url` for remote locations<br /><br />

    when you are successful, a stack id will be printed out like so
    ```
    {
      "StackId" : "arn:aws:cloudformation:ap-southeast-2:<SOME_NUMBERS>:stack/teststack/<SOME_GUID>"
    }
    ```

## Push image to the repository

1. Clone https://github.com/kksy/docker-express-boilerplate

2. Build docker image
    ```
    $ docker build -t docker-express .
    ```

3. Copy the name of the repository URI from Amazon ECR > Repositories in the AWS console.

    e.g. <SOME_NUMBERS>.dkr.ecr.ap-southeast-2.amazonaws.com/test-repository<br /><br />

    There are also instructions when you click "View Push Commands"<br /><br />


4. When the build completes, tag your image
    ```
    docker tag docker-express:latest <REPOSITORY_URI>:latest
    ```

5. Login to ECR using the command below. Copy paste the docker login output to your terminal. You should be able to see 'Login Succeeded'
    ```
    $ aws ecr get-login --no-include-email --region ap-southeast-2
    ```

6. Push image to the repository
    ```
    docker push <REPOSITORY_URI>:latest
    ```

