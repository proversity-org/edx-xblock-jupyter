# edx_xblock_jupyter

XBlock intended to display and manage Jupyter Notebooks.
The Xblock requires the following:

1. A docker deployment of Sifu & JupyterNotebook: https://github.com/proversity-org/edx-api-jupyter
2. Configuration settings in config.yml
3. Access to Django Middleware
3. Creation of Oauth client in the admin backend.
4. Updating Sifu Oauth client details.
4. Studio must be running.

## 1 Docker Deployment
Once the Docker deployment is done, you will need to take note of the
domain name or ip address associated with that deployment.

## 2 Configuration settings
In jupyterhub_xblock/config.yml specify the details from 1:
```yml
'sifu_domain': 'sifu_exmaple.com'
```

## 3 Access to Django Middleware
In order for the xblock to work, it requires access to Django Middleware, secifically crequest:
```py
+INSTALLED_APPS += ('crequest', 'debug_toolbar', 'debug_toolbar_mongo')
 MIDDLEWARE_CLASSES += (
     'django_comment_client.utils.QueryCountDebugMiddleware',
     'debug_toolbar.middleware.DebugToolbarMiddleware',
+    'crequest.middleware.CrequestMiddleware',
 )
```
This must be set in the environment settings.py e.g. lms/envs/devstack.py. Edx currently doesn't support Middleware access in an Xblock but they have taken note of this potential need.

## 4 Oauth Client
Create a new oauth client in the admin backend:
```text
url:           http://sifu_exmaple.com
redirect uri:  http://edx_lms_domain.com
client id:     <generated by admin backend or self prescribed>
client secret: <generated by admin backend or self prescribed>
```
After this is complete, add the client as a 'Trusted Client' as well, in the admin backend.

## 5 Update Sifu
Make the following api call with json data payload:

```POST sifu_exmaple.com/auth/initialize```
```json
{
  'auth_init': {
	url: 'http://sifu_exmaple.com'
	'redirect uri': 'http://edx_lms_domain.com'
	'client id': '<generated by admin backend or self prescribed>'
	'client secret': '<generated by admin backend or self prescribed>'
  },
  'credentials': {
    'username': 'admin',
	'password': 'password'
  }
}
```
This will update the oauth initializer. In future when a docker image is succesfully
deployed, it will create an admin user and credentials, with which you can make this
call, for now they can be left out.

#### Content Security
Update these with the intended hostnames and port numbers, of a frame ancestor.

E.g. A notebook is served in an iframe hosted www.example.com. The Content-Security-Policy would then be:

```py
""" This is for the JupyterHub CSP settings """
c.JupyterHub.tornado_settings = {
    'headers': {
        'Content-Security-Policy': " 'www.example.com:80' "
  }
}
...
""" This is for the JupyterNotebook CSP settings """
c.Spawner.args = ['--NotebookApp.tornado_settings={ \'headers\': { \'Content-Security-Policy\': "\'www.example.com:80\'"}}']
```
## 6 Ensure studio is running
Studio needs to be running in order for Instructors to create the xblock, and upload notebooks. In the lms, the xblock will pull the base notebook and create it remotely in the docker container's volume if it doesn't exist, using sifu api calls. If Studio is not running, and the base notebook does not exist in the docker container's volume, then students will see an empty notebook.

## Use cases
The xblock makes API calls to:
1. Use Edx as an oauth provider, and request an access token from sifu. If this is not a logged in edx user, there will be no joy.
2. Check the existence of the course unit's base notebook file.
2. Upload it if it doesn't exist.
2. Check if the user's course unit notebook exists
3. Create one from the base file if it doesn't exist
4. Request the user's notebook in an Iframe

All API requests to Sifu include the access token provider at the authorization stage.
