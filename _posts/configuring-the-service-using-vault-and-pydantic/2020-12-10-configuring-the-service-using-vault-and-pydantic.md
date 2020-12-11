---
title: Configuring the service using Vault and Pydantic
date: 2020-12-10
modified: 2020-12-10
tags: [python, faust, kafka, vault, sitri]
description: Configuring the service using Vault and Pydantic
external_link: https://dev.to/egnod/configuring-the-service-using-vault-and-pedantic-2d91
external_link_name: DEV.to
---

# Introduction

In this article, I will talk about the configuration for your services using the Vault (KV and so far only the first version, i.e. without versioning secrets) and Pydantic (Settings) under the patronage of [Sitri](https://github.com/LemegetonX/sitri).

So, let's say that we have a *superapp* service with configs set up in Vault and authentication using AppRole, we'll set it up like this (I'll leave the policies setting for access to secret engines and the secrets themselves behind the scenes, since this is quite simple and the article is not about this):

```
Key                        Value
---                        -----
bind_secret_id             true
local_secret_ids           false
policies                   [superapp_service]
secret_id_bound_cidrs      <nil>
secret_id_num_uses         0
secret_id_ttl              0s
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              30m
token_no_default_policy    false
token_num_uses             50
token_period               0s
token_policies             [superapp_service]
token_ttl                  20m
token_type                 default
```
*Note:* naturally, if you have the opportunity and the application goes into production mode, then secret\_id\_ttl is better to do not infinite, setting 0 seconds.

*Superapp* requires configuration of database connection, the connection to kafka and [faust](http://faust.readthedocs.io) configuration for cluster of workers.

At the end, we should have this structure of our test-project:
```bash
super_app
├── config
│ ├── __init__.py
│ ├── provider_config.py
│ ├── faust_settings.py
│ ├── app_settings.py
│ ├── kafka_settings.py
│ └── database_settings.py
├── __init__.py
└── main.py
```

# Bake Sitri provider

Basic library documentation has [a simple example] (https://sitri.readthedocs.io/en/latest/advanced_usage.html#vault-kv-setting-example), configuration via the vault provider, however, it does not cover all the features and can be useful if your application is configured easily enough.

So, first of all, let's configure the vault provider in a file *provider_config.py*:
```python
import hvac  
  
from sitri.providers.contrib.vault import VaultKVConfigProvider  
from sitri.providers.contrib.system import SystemConfigProvider  
  
configurator = SystemConfigProvider(prefix="superapp")  
ENV = configurator.get("env")  
  
  
def vault_client_factory() -> hvac.Client:  
    client = hvac.Client(url=configurator.get("vault_api"))  
  
    client.auth_approle(  
        role_id=configurator.get("role_id"),  
  secret_id=configurator.get("secret_id"),  
  )  
  
    return client  
  
  
provider = VaultKVConfigProvider(  
    vault_connector=vault_client_factory, mount_point=f"{configurator.get('app_name')}/{ENV}"  
)
```

In this code, we get variables from the environment using the system provider to configure the connection to vault, i.e. the following variables must be exported initially:
```bash
export SUPERAPP_ENV=dev
export SUPERAPP_APP_NAME=superapp
export SUPERAPP_VAULT_API=https://your-vault-host.domain
export SUPERAPP_ROLE_ID=<YOUR_ROLE_ID>
export SUPERAPP_SECRET_ID=<YOUR_SECRET_ID>
```
The example assumes that the base mount\_point to your secrets for a specific environment will contain the application name and the environment name, which is why we exported *SUPERAPP\_ENV*. We will define the path to the secrets of individual parts of the application in the settings classes later, so we leave it empty in the *secret\_path* vault provider argument.

# Settings classes
## DBSetting - database connection

To create the settings class, we must use VaultKVSettings as the base class.

File *database_settings.py*:
```python
from pydantic import Field  
  
from sitri.settings.contrib.vault import VaultKVSettings  
  
from superapp.config.provider_config import provider  
  
  
class DBSettings(VaultKVSettings):  
    user: str = Field(..., vault_secret_key="username")  
    password: str = Field(...)  
    host: str = Field(...)  
    port: int = Field(...)  
  
    class Config:  
        provider = provider  
        default_secret_path = "db"
```

As you can see, config data for the database connection is quite simple. This class will by default look at the secret *superapp/dev/db*, as we specified in the *Config* class. At first glance these are simple *pydantic.Field*, but they all have an extra argument *vault\_secret\_key* - it is needed when the key in the secret does not match the name (alias) of the pydantic field in our class, if *vault\_secret\_key* is not specified, the provider will search for the key by the field alias.

For example, in our *superapp*, it is assumed that the *superapp/dev/db* secret has the "password" and "username" keys, but we want the latter to be placed in the "user" field for convenience and brevity.

Let's put the following data in the above secret:
```json
{
  "host": "testhost",
  "password": "testpassword",
  "port": "1234",
  "username": "testuser"
}
```

Now, if we run this code, we will get our secret from vault with our *DBSettings* class:
```python
from superapp.config.database_settings import DBSettings

db_settings = DBSettings()
pprint(db_settings.dict())
# -> 
# {
#     "host": "testhost",
#     "password": "testpassword",
#     "port": 1234,
#     "user": "testuser"
# }
```

## KafkaSettings - connection to brokers

File *kafka_settings.py*:
```python
from typing import Dict, Any  
  
from pydantic import Field  
  
from sitri.settings.contrib.vault import VaultKVSettings  
  
from superapp.config.provider_config import provider, configurator  
  
  
class KafkaSettings(VaultKVSettings):  
    auth_mechanism: str = Field(...)  
    brokers: str = Field(...)  
    auth_data: Dict[str, Any] = Field(...)  
  
    class Config:  
        provider = provider  
        default_secret_path = "kafka"  
        default_mount_point = f"{configurator.get('app_name')}/common"
```

In this case, let's imagine that there is one kafka instance for different environments of our service, so the secret is stored along the path *superapp/common/kafka*:

*Note:* You can also set secret\_path or/and mount\_point at the field level so that the provider requests specific values from different secrets (if required). Here is a quote with prioritization of the secret path and mount point from the [documentation](https://sitri.readthedocs.io/en/latest/advanced_usage.html#vault-settings-configurators):

> Secret path prioritization:
>1.  vault\_secret\_path (Field arg)
>2.  default\_secret_path (Config class field)
>3.  secret\_path (provider initialization optional arg)

>Mount point prioritization:
>1.  vault\_mount\_point (Field arg)
>2.  default\_mount\_point (Config class field)
>3.  mount\_point (provider initialization optional arg)

```json
{
  "auth_data": "{\"password\": \"testpassword\", \"username\": \"testuser\"}",
  "auth_mechanism": "SASL_PLAINTEXT",
  "brokers": "kafka://test"
}
```
Or
```json
{
    "auth_data":
    {
        "password": "testpassword",
        "username": "testuser"
    },
    "brokers": "kafka://test",
    "auth_mechanism": "SASL_PLAINTEXT"
}
```
*Note:* VaultKVSettings can understand both json and Dict itself.

As a result, this class of settings will be able to collect data from the secret like this:
```python
{
    "auth_data":
    {
        "password": "testpassword",
        "username": "testuser"
    },
    "brokers": "kafka://test",
    "auth_mechanism": "SASL_PLAINTEXT"
}
```

## FaustSettings - global configure faust and individual agents

File *faust_settings.py*:
```python
from typing import Dict  
  
from pydantic import Field, BaseModel  
  
from sitri.settings.contrib.vault import VaultKVSettings  
  
from superapp.config.provider_config import provider  
  
  
class AgentConfig(BaseModel):  
    partitions: int = Field(...)  
    concurrency: int = Field(...)  
  
  
class FaustSettings(VaultKVSettings):  
    app_name: str = Field(...)  
    default_partitions_count: int = Field(..., vault_secret_key="partitions_count")  
    default_concurrency: int = Field(..., vault_secret_key="agent_concurrency")  
    agents: Dict[str, AgentConfig] = Field(default=None, vault_secret_key="agents_specification")  
  
    class Config:  
        provider = provider  
        default_secret_path = "faust"
```

Secret *superapp/dev/faust*:
```json
{
  "agent_concurrency": "5",
  "app_name": "superapp-workers",
  "partitions_count": "10"
}
```

If our secret is written as indicated above, then individual agents will take information about the number of partitions in their topics and concurrency from the default fields. 
```python
{
  "agents": None,
  "app_name": "superapp-workers",
  "default_concurrency": 5,
  "default_partitions_count": 10
}
```

However, with the help of the *AgentConfig* model, we can set individual values for a specific agent. For example, if we have agent *X* with 5 partitions and concurrency 2, then we can change our secret so that information about this agent is in the "agents" field.

Secret *superapp/dev/faust*:
```json
{
  "agent_concurrency": "5",
  "agents_specification": {
    "X": {
      "concurrency": "2",
      "partitions": "5"
    }
  },
  "app_name": "superapp-workers",
  "partitions_count": "10"
}
```

If we initialize our settings now, we will receive, in addition to the default fields, also the configuration for a specific agent:
```python
{
    "agents":
    {
        "X":
        {
            "concurrency": 2,
            "partitions": 5
        }
    },
    "app_name": "superapp-workers",
    "default_concurrency": 5,
    "default_partitions_count": 10
}
```

## Combine the settings classes into a single configuration model

To make our configs even more convenient to use, let's combine all our settings classes into one model by applying the default_factory:

File *app_settings.py*:
```python
from pydantic import BaseModel, Field  
  
from superapp.config.database_settings import DBSettings  
from superapp.config.faust_settings import FaustSettings  
from superapp.config.kafka_settings import KafkaSettings  
  
  
class AppSettings(BaseModel):  
    db: DBSettings = Field(default_factory=DBSettings)  
    faust: FaustSettings = Field(default_factory=FaustSettings)  
    kafka: KafkaSettings = Field(default_factory=KafkaSettings)
```

Now, let's go to the *main.py* file and test the collection of complete configuration data for our application:
```python
from superapp.config import AppSettings  
  
config = AppSettings()  
  
print(config)  
print(config.dict())
```

We get the output of application configuration:
```python
db=DBSettings(user='testuser', password='testpassword', host='testhost', port=1234) 
faust=FaustSettings(app_name='superapp-workers', default_partitions_count=10, default_concurrency=5, agents={'X': AgentConfig(partitions=5, concurrency=2)}) 
kafka=KafkaSettings(auth_mechanism='SASL_PLAINTEXT', brokers='kafka://test', auth_data={'password': 'testpassword', 'username': 'testuser'})
```
```python
{
    "db":
    {
        "host": "testhost",
        "password": "testpassword",
        "port": 1234,
        "user": "testuser"
    },
    "faust":
    {
        "agents":
        {
            "X":
            {
                "concurrency": 2,
                "partitions": 5
            }
        },
        "app_name": "superapp-workers",
        "default_concurrency": 5,
        "default_partitions_count": 10
    },
    "kafka":
    {
        "auth_data":
        {
            "password": "testpassword",
            "username": "testuser"
        },
        "brokers": "kafka://test",
        "auth_mechanism": "SASL_PLAINTEXT"
    }
}
```
Happiness, joy, delight!

# Afterword

As you can see, the configuring is quite simple with Sitri, after it we get a clear configuration scheme with the required data types for the values, even if they were stored in strings in the vault by default.

Write comments about lib, code or general impressions. I will be glad to any feedback!

P.S. [I have uploaded the code from the article to github] (https://github.com/Egnod/article_sitri_vault_pydantic)
