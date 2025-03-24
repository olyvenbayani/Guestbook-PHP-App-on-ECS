# Deploying a PHP Guestbook Application on Amazon ECS

This guide walks you through creating, containerizing, and deploying a PHP Guestbook application using Docker and Amazon ECS. The app allows users to submit messages that are displayed on the page, stored in a `messages.txt` file. You'll push the Docker image to Amazon ECR and deploy it on an ECS cluster with Fargate, making it accessible via a public IP.

## Prerequisites
- AWS CLI installed and configured (`aws configure`).
- Docker installed and running.
- An existing ECS cluster (e.g., `cluster-<yourname>`).
- A subnet and security group with port 80 open to the public.
- IAM permissions for ECS, ECR, and CloudWatch Logs.

## Step 1: Create the PHP Application

1. **Create a Project Directory**
   Open a terminal and run:
   ```bash
   mkdir php-guestbook
   cd php-guestbook

2. **Create a file named index.php with the following content:**

```php
<?php
$messages = file_exists('messages.txt') ? file('messages.txt', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) : [];
if ($_SERVER['REQUEST_METHOD'] === 'POST' && !empty($_POST['message'])) {
    file_put_contents('messages.txt', $_POST['message'] . PHP_EOL, FILE_APPEND);
    header('Location: /');
    exit;
}
?>
<!DOCTYPE html>
<html>
<head><title>PHP Guestbook</title></head>
<body>
    <h1>Guestbook</h1>
    <form method="POST">
        <input type="text" name="message" placeholder="Enter your message" required>
        <button type="submit">Submit</button>
    </form>
    <h2>Messages:</h2>
    <ul>
        <?php foreach ($messages as $msg) { echo "<li>" . htmlspecialchars($msg) . "</li>"; } ?>
    </ul>
</body>
</html>
```

## Step 2: Create a Dockerfile

1. Create Dockerfile In the php-guestbook directory, create a file named Dockerfile with:

```
FROM php:7.4-apache
WORKDIR /var/www/html
COPY index.php .
RUN chown www-data:www-data /var/www/html -R && chmod 755 /var/www/html
EXPOSE 80
```

## Step 3: Build and Push the Docker Image to ECR
1. Set Environment Variables Replace <yourname> and <aws-account-id> with your values:

```
export DOCKER_REPO="php-guestbook-<yourname>"
export ECR_REPO="php-guestbook-<yourname>"
export AWS_ACCOUNT_ID="<aws-account-id>"
export AWS_REGION="ap-southeast-1"
```

2. Build the Docker Image Run:

```docker build -t $DOCKER_REPO .```

3. Create an ECR Repository (if it doesn’t exist)
```
aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION
```

4. Tag and Push the Image

Tag the image:

```
docker tag $DOCKER_REPO:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
```

Authenticate Docker with ECR:

```
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

Push the image:

```
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
```

## Step 4: Update the ECS Task Definition
1. Create task-definition.json In the php-guestbook directory, create task-definition.json with:

```
{
  "family": "php-guestbook-task-<yourname>",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::<aws-account-id>:role/ECSExecutionRole",
  "containerDefinitions": [
    {
      "name": "php-guestbook-container",
      "image": "<aws-account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/php-guestbook-<yourname>:latest",
      "memory": 256,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "<yourname>-guestbook-logs",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
```

2. Register the Task Definition
```
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

## Step 5: Deploy and Test

1. Run the Task Replace <your-subnet-id> and <your-security-group-id>
```
aws ecs run-task --cluster <cluster_name> --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[<your-subnet-id>],securityGroups=[<your-security-group-id>],assignPublicIp=ENABLED}" --task-definition php-guestbook-task-<yourname>
```


2. Find the Public IP Go to the AWS ECS Console. Navigate to your cluster (cluster-<yourname>). Under "Tasks," find the running task and note its public IP.

3. Test the Application: Open http://<public-ip> in a browser.

4. Submit a message (e.g., "Hello, ECS!") and verify it appears in the list.

