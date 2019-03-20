# k8-init
Kubernetes Init Scripts

This is a collection of scripts that I have used to initialize POD environments. I have tried to ensure that they all use a similar invocation and configuration mechanism that is documented below.

## Configuration

Configuration happens via Configmap that needs to be loaded before the init container runs. The configmap is pulled in as environment variables and has the following format. 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: init.env
data:
  POSTGRES_DATABASE: "Application database name"
  SSM_POSTGRES_HOSTNAME: "/path/to/postgres/hostname/in/ssm"
  SSM_POSTGRES_USER: "/path/to/application/database/user/in/ssm"
  SSM_POSTGRES_USER_PASSWORD: "/path/to/application/user/password/in/ssm"
  SSM_POSTGRES_ROOT_USER: "/path/to/database/root/user/in/ssm"
  SSM_POSTGRES_ROOT_USER_PASSWORD: "/path/to/database/root/user/password/in/ssm"
  SSM_EXAMPLE_PARAMETER: "/ssm/parameter/path"
  NON_SSM_PARAMETER: "<parameter>"
``` 

Any parameter that starts with SSM_POSTGRES is used to configure the application database, this means that a database and a user are created and the ownership is shifted tothe created user. 

If a parameter starts with SSM_, it is simply pulled from SSM and put into the created secret. 

If a parameter does not start with SSM, it is not inserted into the secret, this is generally used to modify the behaviour of the script. 

## Database initilization
The database initilization script has 6 required parameters, they are
```
POSTGRES_DATABASE
SSM_POSTGRES_HOSTNAME
SSM_POSTGRES_USER
SSM_POSTGRES_USER_PASSWORD
SSM_POSTGRES_ROOT_USER
SSM_POSTGRES_ROOT_USER_PASSWORD
```
The POSTGRES_DATABASE variable is the name of the database to be operated on, it should not be the default database (usually postgres), this is the only required parameter that should not be an SSM path. 
The SSM_POSTGRES_USER/PASSWORD variables are the users which are intended to be used by the application. The database created will be owned by this user. These credentials are used to construct the connection strings requested. 
The SSM_POSTGRES_ROOT_USER/PASSWORD variables are the root credentials for the database host, these are used to create and modify the database and roles. They do not appear in the secret created. 

In addition, there are two optional parameters
```
ALLOW_CHOWN_DATABASE
ALLOW_DROP_DATABASE
```
The ALLOW_CHOWN_DATABASE is a boolean and is used if the database already exists and is owned by another user. This will create the role, then chown all of the tables to the role created. 

The ALLOW_DROP_DATABASE command will drop a database if it exists before attempting to create one.

When the script runs, it first checks if the database exists, if it does, it attempts to login with the user and password provided. If this succeeds, then all is well and the database initialization is successful. 
If the login attempt fails, but the database exists, the exit is unsuccessful. This is because we do not want to alter the database unless the user has explicitly given permission. If you have data that you want to keep in the database, using ALLOW_CHOWN_DATABASE will allow you to change the user. This will revoke access to any existing users. Otherwise, using ALLOW_DROP_DATABASE will drop the database if it exists. 

## SSM

SSM is an AWS service called Systems Manager, specifically what we are using is called parameter store. It allows the storage of parameters including SecureStrings which are encrypted by KMS. 
