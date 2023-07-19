# JupyterHub Configuration Example

With running JupyterHub there are many configuration combinations you may choose to use. You will learn about a few common configurations that could valuable for your configurations.

- AWS Fargate Spawner
- Named Servers with API Service
- Spawner Callbacks
- Entra ID Authentication (Azure AD Authentication)

## Prerequisite

As a matter of good practice you won't want your secrets in your configuration file. You will want to include a helper function to get a secret from your secret vault. Below we have an example with AWS Secrets Manager that you can use with your `jupyterhub_config.py` file. This function is used by a couple of the following sections.

```python
import json
import boto3
from botocore.exceptions import ClientError
def get_secret(secret, key):
    secret_name = secret
    region_name = "us-east-1"
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    try:
        get_secret_value_response = client.get_secret_value(
            SecretId=secret_name
        )
    except ClientError as e:
        raise e
    secret = get_secret_value_response['SecretString']
    val = json.loads(secret)
    return val[key]
```

## AWS Fargate Spawner

Your JupyterHub instance can spawn Jupyter Notebook instances as AWS Fargate Tasks. This open source [Fargate Spawner](https://github.com/uktrade/fargatespawner) provides a way to interface with the AWS APIs to create new tasks.

You can install the package to your JupyterHub Docker image as shown below.

```docker
RUN pip3 install fargatespawner
```

Then within your `jupyterhub_config.py` file you will need to specify how you would like the spawner to create the tasks. Then finally you will want to configure how the spawner will authenticate with the AWS APIs.

```python
from fargatespawner import FargateSpawner
c.JupyterHub.spawner_class = FargateSpawner
c.FargateSpawner.aws_region = 'us-east-1'
c.FargateSpawner.aws_ecs_host = 'ecs.us-east-1.amazonaws.com'
c.FargateSpawner.notebook_port = 8888
c.FargateSpawner.notebook_scheme = 'http'
c.FargateSpawner.get_run_task_args = lambda spawner: {
    'cluster': 'jupyterLabs',
    'taskDefinition': 'JupyterLab:2',
    'overrides': {
        'containerOverrides': [{
            'command': spawner.cmd,
            'environment': [
                {
                    'name': name,
                    'value': value,
                } for name, value in spawner.get_env().items()
            ],
            'name': 'dotnet',
        }],
    },
    'count': 1,
    'launchType': 'FARGATE',
    'networkConfiguration': {
        'awsvpcConfiguration': {
            'assignPublicIp': 'ENABLED',
            'securityGroups': ['sg-012345678abcd0123'],
            'subnets':  ['subnet-abcd0123','subnet-0123abcd'],
        },
    },
}

from fargatespawner import FargateSpawnerECSRoleAuthentication
c.FargateSpawner.authentication_class = FargateSpawnerECSRoleAuthentication
```

## Named Servers with API Service

You may want to use named instances that can be spawned independent of a user interaction. To accomplish this you can allow named servers, setup a spawner with admin permissions that uses an API secret from your secret vault. These can be defined in your `jupyterhub_config.py` file.

```python
c.JupyterHub.allow_named_servers = True

c.JupyterHub.services = [
    {
        "name": "spawner-admin",
        "api_token": get_secret('JupyterHubService','api_token')
    },
]

c.JupyterHub.load_roles = [
    {
        "name": "service-role",
        "scopes": [
            "admin:users",
            "admin:servers",
        ],
        "services": [
            "spawner-admin",
        ],
    }
]
```

## Spawner Callbacks

When you want to run JupyterHub itself as a container instance, the configuration needs a little more guidance for how the Notebook instances should communicate with it. The below `jupyterhub_config.py` file snippet requires you to set most notably the `c.Spawner.hub_connect_url` to the URL of your JupyterHub instance which the Jupyter Notebook instance will communicate with.

```python
c.Spawner.ip = '0.0.0.0'
c.Spawner.port = 8888
c.Spawner.env_keep = ['PYTHONPATH', 'CONDA_ROOT', 'CONDA_DEFAULT_ENV', 'VIRTUAL_ENV', 'LANG', 'LC_ALL', 'JUPYTERHUB_SINGLEUSER_APP']
c.Spawner.hub_connect_url = 'https://jupyter.your.domain.com'
c.Spawner.start_timeout = 120
```

## Entra ID (Azure AD) Authentication

Enabling single sign-on with your existing Identity Provider (IdP) simplifies the experience for the users of your JupyterHub instance and the administrator burden as well. For this setup you will use Microsoft Entra ID Authentication, formerly known as Azure AD Authentication.

You can install the prerequisite Python packages in your JupyterHub Docker image as shown below.

```docker
RUN pip3 install PyJWT
RUN pip3 install oauthenticator
```

With your prerequisites installed you can configure your `jupyterhub_config.py` file to utilize the [OAuthenticator](https://oauthenticator.readthedocs.io/en/latest/tutorials/provider-specific-setup/providers/azuread.html) package. You will need to specify your Entra ID Tenant ID (Azure AD Tenant ID) GUID for the `c.AzureAdOAuthenticator.tenant_id` property. Specify your JupyterHub callback URL for the authentication flow to return your users to with the `c.AzureAdOAuthenticator.oauth_callback_url` property. The Client ID of your Entra ID Application with the `c.AzureAdOAuthenticator.client_id` property. Then finally make sure your Client Secret is within your secret vault.

```python
from oauthenticator.azuread import AzureAdOAuthenticator
c.JupyterHub.authenticator_class = AzureAdOAuthenticator

c.AzureAdOAuthenticator.tenant_id = '0123abcd-01ab-ab01-abcd-0123abcd4567'

c.AzureAdOAuthenticator.oauth_callback_url = 'https://jupyter.your.domain.com/hub/oauth_callback'
c.AzureAdOAuthenticator.client_id = 'abcd0123-ab01-01ab-dcba-45670123abcd'
c.AzureAdOAuthenticator.client_secret = get_secret('JupyterHub', 'client_secrect')
c.AzureAdOAuthenticator.scope = ['openid']
```

## Summary

Through this post you saw how to setup:

- A helper function to pull secrets from AWS Secrets Manager
- Configure JupyterHub to use an AWS Fargate Spawner
- Configure JupyterHub to allow Named Servers with an API Service
- Configure Spawner Callbacks to a dynamic JupyterHub instance
- Configure JupyterHub to use Entra ID Authentication (Azure AD Authentication)

There is plenty more configuration you can do with JupyterHub and the Jupyter ecosystem in general.
