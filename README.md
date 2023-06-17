# Migrating a Flask Web App to Microsoft Azure

## Introduction
In this project, a flask web app is migrated to Microsoft Azure. The HTML and frontend python code is containerized with Docker, uploaded to Azure Container Registry (ACR), and deployed to Azure Kubernetes Service (AKS). The backend is deployed to a Linux VM in Azure, and the local database is replaced with an Azure SQL database. Additional features are implemented into the web app using Microsoft Cognitive Services. Namely, the Microsoft Translator API will be used to translate user entries and store them in the database, and the Content Moderation API will be used to prevent personally identifiable information (PII) from being added to the to-do list. The purpose of this lab was to explore the various capabilities of cloud computing and the offerings available in Microsoft Azure. 

#

<details>
<summary>
  
## Creating the VM and using SCP to transfer files
  
  </summary>  
<br />
The first stage of this lab was to create the virtual machine. From the Azure Portal, the virtual machine was created with the following configuration:
  
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/0136788f-4479-4fb3-8848-431538ad86cf)

  
  After creating the VM, port 22 was enabled in the Networking settings, which allows us to SSH and SCP files into the VM. 
  
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/af0c1fca-ac02-4e45-bf20-6929fccf838e)

  
  
  I locally accessed it via the command line on a local Windows machine (the public IP can be found in the networking section of the VM settings page):
  
 ```
  ssh azureuser@[public_IP]
```
  
  Now, the flask app files are copied to Azure
  
  ```
  scp -r C:\Users\Admin\Desktop\FlaskAppFiles azureuser@[public_IP]:/home/azureuser/myapp
  ```
  </details>
  
  #

<details>
<summary>
  
## Implementing the Microsoft Translator API
  
  </summary>  
<br />
Now, the Microsoft Translator API is implemented into the backend python code. First, the Microsoft Translator resource must be created, which was done from the portal. The configuration is as follows:
  
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/1b09ad67-dbe3-4b4c-9c15-0611227c3350)

The API keys will be provided in the 'Keys and Endpoint' section of the resource page.
  
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/0942fb52-fad6-49bb-ab06-7d261c6774d1)

Before the item is stored in the database, I will use the Microsoft Translator API to translate the item into Spanish. The translated text will then be concatenated with the original entry. The resulting string will then be inserted into the database. To implement this, we must modify the backend code todolist_api.py. The code modification is as follows:

  
  <details>
    <summary> <strong> Modified Backend Code with Microsoft Translator Implemented </strong> </summary>
<br>
    
```
# RESTful API
from flask import Flask, render_template, redirect, g, request, url_for, jsonify, Response
from flask_cors import CORS
import sqlite3
import urllib
import json
import requests
import uuid
import json

DATABASE = 'todolist.db'

app = Flask(__name__)
CORS(app)
app.config.from_object(__name__)

# Function to call Microsoft Translator API and retrieve translated text
def translate_text(text, target_language):
    key = "[API_KEY]"
    endpoint = "https://api.cognitive.microsofttranslator.com"

    # Location, also known as region.
    # It can be found in the Azure portal on the Keys and Endpoint page.
    location = "global"

    path = '/translate'
    constructed_url = endpoint + path

    params = {
        'api-version': '3.0',
        'from': 'en',
        'to': 'es'
    }

    headers = {
        'Ocp-Apim-Subscription-Key': "[API_KEY]",
        'Ocp-Apim-Subscription-Region': "global",
        'Content-type': 'application/json',
        'X-ClientTraceId': str(uuid.uuid4())
    }
   
   body = [{
     'text': text 
    }]

    # Make the API call to translate the text
    response = requests.post(constructed_url, params=params, headers=headers, json=body)
    translation_data = response.json()

    # Extract the translated text from the API response
    translated_text = translation_data[0]['translations'][0]['text']

    return translated_text 

@app.route("/api/items")  # default method is GET
def get_items():
    db = get_db()
    cur = db.execute('SELECT what_to_do, due_date, status FROM entries')
    entries = cur.fetchall()
    tdlist = [dict(what_to_do=row[0], due_date=row[1], status=row[2])
              for row in entries]
    response = Response(json.dumps(tdlist),  mimetype='application/json')
    return response

@app.route("/api/items", methods=['POST'])
def add_item():
    db = get_db()
    what_to_do = request.json['what_to_do']
    due_date = request.json['due_date']
    translated_text = translate_text(what_to_do, 'es')

    # Concatenate the original and translated text
    concatenated_text = f'{what_to_do} - {translated_text}'
    db.execute('insert into entries (what_to_do, due_date) values (?, ?)',
               [concatenated_text, due_date])
    db.commit()
    return jsonify({"result": True})

@app.route("/api/items/<item>", methods=['DELETE'])
def delete_item(item):
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("DELETE FROM entries WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})

@app.route("/api/items/<item>", methods=['PUT'])
def update_item(item):
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("UPDATE entries SET status='done' WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})

def get_db():
    """Opens a new database connection if there is none yet for the
    current application context.
    """
    if not hasattr(g, 'sqlite_db'):
        g.sqlite_db = sqlite3.connect(app.config['DATABASE'])
    return g.sqlite_db

@app.teardown_appcontext
def close_db(error):
    """Closes the database again at the end of the request."""
    if hasattr(g, 'sqlite_db'):
        g.sqlite_db.close()

if __name__ == "__main__":
    app.run("0.0.0.0", port=5001)
  ```
    
</details>
    </details>
  

  
  #

    
    
<details>
<summary>
  
## Implementing the Content Moderator API
  
  </summary>  
<br />
The implementation of the Content Moderator is similar to the implementation of the Microsoft Translator. Search for the Content Moderator in the searchbar of the Azure Portal. Once the service is created, the API keys will be provided in the 'Keys and Endpoint' section of the resource page.
  
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/b21a819d-7d1d-45be-8966-c646e7b87ed5)

  Once again, the backend code is modified to implement the content moderator. The function check_content_moderation() will use the Content Moderator API. The granularity of the Content Moderator is also shown, as we will use add logic in check_content_moderation() to only block PII such as emails and phone numbers. Some logic will also be added to the add_item() function to ensure the input doesn't have any PII.
  
   <details>
    <summary> <strong> Modified Backend Code with Microsoft Content Moderator Implemented </strong> </summary>
<br>
    
```
# RESTful API
from flask import Flask, render_template, redirect, g, request, url_for, jsonify, Response
from flask_cors import CORS
import sqlite3
import urllib
import json
import requests
import uuid
import json

DATABASE = 'todolist.db'

app = Flask(__name__)
CORS(app)
app.config.from_object(__name__)

# Function to call Microsoft Translator API and retrieve translated text
def translate_text(text, target_language):
    key = "[API_KEY]"
    endpoint = "https://api.cognitive.microsofttranslator.com"

    # Location, also known as region.
    # It can be found in the Azure portal on the Keys and Endpoint page.
    location = "global"

    path = '/translate'
    constructed_url = endpoint + path

    params = {
        'api-version': '3.0',
        'from': 'en',
        'to': 'es'
    }

    headers = {
        'Ocp-Apim-Subscription-Key': "[API_KEY]",
        'Ocp-Apim-Subscription-Region': "global",
        'Content-type': 'application/json',
        'X-ClientTraceId': str(uuid.uuid4())
    }
   
   body = [{
     'text': text 
    }]

    # Make the API call to translate the text
    response = requests.post(constructed_url, params=params, headers=headers, json=body)
    translation_data = response.json()

    # Extract the translated text from the API response
    translated_text = translation_data[0]['translations'][0]['text']

    return translated_text 

# Use the Content Moderator API to check the text
def check_content_moderation(text):
    key = "[API_KEY]"
    endpoint = "https://[Resource_name].cognitiveservices.azure.com/"

    path = '/contentmoderator/moderate/v1.0/ProcessText/Screen'
    constructed_url = endpoint + path

    headers = {
        'Content-Type': 'text/plain',
        'Ocp-Apim-Subscription-Key': key,
        'Ocp-Apim-Subscription-Region': 'eastus',
    }

    params = {
        'classify': 'True'
    }

    # Make the API call to Content Moderator
    response = requests.post(constructed_url, headers=headers, params=params, data=text)
    moderation_data = response.json()
    return moderation_data

def has_pii(response):
    return bool(response.get('PII'))


@app.route("/api/items")  # default method is GET
def get_items(): 
    db = get_db()
    cur = db.execute('SELECT what_to_do, due_date, status FROM entries')
    entries = cur.fetchall()
    tdlist = [dict(what_to_do=row[0], due_date=row[1], status=row[2])
              for row in entries]
    response = Response(json.dumps(tdlist),  mimetype='application/json')
    return response

	
# Check the content via the Content Moderator
@app.route("/api/items", methods=['POST'])
def add_item(): 
    db = get_db()
    what_to_do = request.json['what_to_do']
	response = check_content_moderation(what_to_do)
    if has_pii(response):
        return jsonify({"result": False})  

    due_date = request.json['due_date']
    translated_text = translate_text(what_to_do, 'es')

    # Concatenate the original and translated text
    concatenated_text = f'{what_to_do} - {translated_text}'
    db.execute('insert into entries (what_to_do, due_date) values (?, ?)',
               [concatenated_text, due_date])
    db.commit()
    return jsonify({"result": True})

@app.route("/api/items/<item>", methods=['DELETE'])
def delete_item(item): 
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("DELETE FROM entries WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})

@app.route("/api/items/<item>", methods=['PUT'])
def update_item(item): 
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("UPDATE entries SET status='done' WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})

def get_db():
    """Opens a new database connection if there is none yet for the
    current application context.
    """
    if not hasattr(g, 'sqlite_db'):
        g.sqlite_db = sqlite3.connect(app.config['DATABASE'])
    return g.sqlite_db

@app.teardown_appcontext
def close_db(error):
    """Closes the database again at the end of the request."""
    if hasattr(g, 'sqlite_db'):
        g.sqlite_db.close()

if __name__ == "__main__":
    app.run("0.0.0.0", port=5001)
  ```
    
</details>
    
</details>
     

  #
    
<details>
<summary>
  
## Creating the Dockerfile and Uploading to Azure Container Registry 
  
  </summary>  
<br />
In this project, the front end will be containerized and ultimately deployed on Azure Kubernetes Service. Containers are much more portable and are lightweight compared to using VMs. By deploying our container to AKS, we can create an applicable that is easily scalable and have access to features such as load balancing. While our application is relatively simple in this instance, these services grant scalability, flexibility, high availability, and simplified management.
    
  
  First, we need to build the dockerfile. For this project it was done in the VM itself. Have the templates folder (index.html is stored here) and todolist.py together in a folder on the VM.
  A separate file named 'Dockerfile' was created with the following instructions:
  
  ```
FROM python:3
RUN pip install flask
RUN pip install requests
EXPOSE 5000/tcp
COPY todolist.py 
COPY templates/index.html templates/  
  ```
  
Once this Dockerfile is saved, we can use the following command to build:
  
  ```
  docker build -t theazfront .
```
  
Testing can be done with:
  
```
docker run -p 5000:5000 theazfront
```
  
  
  
The application should now be reachable by accessing [VM_PUBLIC_IP]:5000 in the address bar. 
  
Now we will create a Container Registry in Azure. Just like other resources on Azure, the portal may be used. I navigated to Create a Resource, and under Containers clicked ‘Create a Container Registry’.
  
  After naming and creating the registry, I tagged the Docker image with the login server of my ACR instance (named ciscfrontcontainer) with:
  
  ```
docker tag theazfront ciscfrontcontainer.azurecr.io/theazfront:latest
```

Now I can login to ACR:

```
az acr login --name ciscfrontcontainer
```
  
  Next, I push the Docker image to my ACR instance with the Docker CLI:

  ```
  docker push ciscfrontcontainer.azurecr.io/theazfront:latest 
```

 </details>
     
    
     

  #

    
    
<details>
<summary>
  
## Deploying the Container to Azure Kubernetes Service (AKS) 
  
  </summary>  
<br />
Now that we have containerized our frontend, we will have it deployed to Azure Kubernetes Service. We can create the AKS cluster via the CLI with:
  
  ```
  az aks create --resource-group NewerResourceGroup --name theclustercisc --node-count 2 --generate-ssh-keys --attach-acr ciscfrontcontainer
```

Now, the YAML files will be created. The YAML file defines our deployment, and is useful as this format makes it easier to manage, maintain, and automate deployments in a consistent manner.
We will need two YAML files, a Deployment YAML and a Service YAML. These YAML files can be created in either the local machine or in the VM that we made. For this project, they were written locally.
  
  The deployment YAML's code is as follows:

```
  apiVersion: apps/v1
kind: Deployment
metadata:
  name: todolist-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todolist-app
  template:
    metadata:
      labels:
        app: todolist-app
    spec:
      containers:
      - name: todolist-app
        image: ciscfrontcontainer.azurecr.io/theazfront:latest
        ports:
        - containerPort: 5000
```
  
 The service YAML is as follows:

```
apiVersion: v1
kind: Service
metadata:
  name: todolist-service
spec:
  type: LoadBalancer
  selector:
    app: todolist-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  ```

  Then, I navigated to the folder where these files were located and used the following to deploy the YAML files:
  
```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Now, we can retrieve the external IP of our Kubernetes cluster by using the following command:

```
kubectl get services
```

Now, the web app was successful accessed by entering that IP in the address bar. 
  
 </details>
    
  #

    
    
<details>
<summary>
  
## Setting up the Azure SQL Database 
  
  </summary>  
<br />
In this step, the todolist.db file will be replaced with an Azure SQL Database. To accomplish this I went to Azure Marketplace and Created ‘SQL Database’. I had to create a server for this (the option came up in the portal). I set up a server admin user and password. The configuration is as shown:

![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/f7557f93-1849-4cdd-95a2-a65672ccf283)

	
To use the Azure database, we will have to make some modifications in the backend code. First, install required dependencies:
	
```
pip install pyodbc 
sudo apt-get install unixODBC
sudo -H pip install pyodbc
```

Then I installed Microsoft ODBC on the machine by following the instructions in the [SQL Server Documentation](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver16&tabs=ubuntu18-install%2Cubuntu17-install%2Cdebian8-install%2Credhat7-13-install%2Crhel7-offline).

In the query editor of the database's page, the following query is used to create a table:
	
```
IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'entries')
BEGIN
    CREATE TABLE entries (
        what_to_do VARCHAR(255),
        due_date VARCHAR(255),
        status VARCHAR(255)
    )
END
```
Finally, the todolist_api.py file in the Linux VM may be edited to use the database:
	   <details>
    <summary> <strong> Modified Backend Code with Azure SQL Database implemented </strong> </summary>
<br>
    
```
# RESTful API
from flask import Flask, render_template, redirect, g, request, url_for, jsonify, Response
from flask_cors import CORS
import sqlite3
import urllib
import json
import requests
import uuid
import json
import pyodbc

app = Flask(__name__)
CORS(app)
app.config.from_object(__name__)

# Function to call Microsoft Translator API and retrieve translated text
def translate_text(text, target_language):
    key = "[API_KEY]"
    endpoint = "https://api.cognitive.microsofttranslator.com"

    # Location, also known as region.
    # It can be found in the Azure portal on the Keys and Endpoint page.
    location = "global"

    path = '/translate'
    constructed_url = endpoint + path

    params = {
        'api-version': '3.0',
        'from': 'en',
        'to': 'es'
    }

    headers = {
        'Ocp-Apim-Subscription-Key': "[API_KEY]",
        'Ocp-Apim-Subscription-Region': "global",
        'Content-type': 'application/json',
        'X-ClientTraceId': str(uuid.uuid4())
    }

    body = [{
        'text': text
    }]

    # API call to translate the text
    response = requests.post(constructed_url, params=params, headers=headers, json=body)
    translation_data = response.json()

    # Extract the translated text 
    translated_text = translation_data[0]['translations'][0]['text']

    return translated_text
def check_content_moderation(text):
    key = "[API_KEY]"
    endpoint = "https://[Resource_Name].cognitiveservices.azure.com/"

    path = '/contentmoderator/moderate/v1.0/ProcessText/Screen'
    constructed_url = endpoint + path

    headers = {
        'Content-Type': 'text/plain',
        'Ocp-Apim-Subscription-Key': key,
        'Ocp-Apim-Subscription-Region': 'eastus',
    }

    params = {
        'classify': 'True'
    }

    # Make the API call to Content Moderator
    response = requests.post(constructed_url, headers=headers, params=params, data=text)
    moderation_data = response.json()
    return moderation_data

def has_pii(response):
    return bool(response.get('PII'))

@app.route("/api/items")  # default method is GET
def get_items():
    db = get_db()
    cur = db.execute('SELECT what_to_do, due_date, status FROM entries')
    entries = cur.fetchall()
    tdlist = [dict(what_to_do=row[0], due_date=row[1], status=row[2])
              for row in entries]
    response = Response(json.dumps(tdlist),  mimetype='application/json')
    return response


@app.route("/api/items", methods=['POST'])
def add_item():  
    db = get_db()
    what_to_do = request.json['what_to_do']
    response = check_content_moderation(what_to_do)
    if has_pii(response):
        return jsonify({"result": False})
    due_date = request.json['due_date']
    translated_text = translate_text(what_to_do, 'es')

    # Concatenate the original and translated text
    concatenated_text = f'{what_to_do} - {translated_text}'
 
    db.execute('insert into entries (what_to_do, due_date) values (?, ?)',
               [concatenated_text, due_date])
    db.commit()
    return jsonify({"result": True})


@app.route("/api/items/<item>", methods=['DELETE'])
def delete_item(item): 
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("DELETE FROM entries WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})

@app.route("/api/items/<item>", methods=['PUT'])
def update_item(item): 
    item = urllib.parse.unquote(item)
    db = get_db()
    db.execute("UPDATE entries SET status='done' WHERE what_to_do='"+item+"'")
    db.commit()
    return jsonify({"result": True})
	
# Utilize Azure SQL Database
def get_db():
    if not hasattr(g, 'sql_db'):
        # Configure the Azure SQL Database connection details
        server = '[server_name].database.windows.net'
        database = 'todolistdatabase'
        username = '[username]'
        password = '{[pasword]}'
        driver = '{ODBC Driver 17 for SQL Server}'
        
        # Create the connection string
        connection_string = f"DRIVER={driver};SERVER={server};DATABASE={database};UID={username};PWD={password}"
        
        # Establish a connection
        g.sql_db = pyodbc.connect(connection_string)
    return g.sql_db

@app.teardown_appcontext
def close_db(error):
    """Closes the database again at the end of the request."""
    if hasattr(g, 'sql_db'):
        g.sql_db.close()
if __name__ == "__main__":
    app.run("0.0.0.0", port=5001)
  ```
		   
		   
</details>
	

 </details>
		   
    
#

<details>
<summary>
  
## App Demonstration 
  
  </summary>  
<br />
New items are added to the Todo List by pressing the 'add a new item' button. Since we are also utilizing the content moderator API, if a user tries to enter PII such as a phone number or email, it will not be added to the list if they try to save the item. 



![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/e494e19d-ea1a-40c5-8356-ab85e4b6857d)



When a new entry is added, it is being translated by the Translator API in the backend todolist_api.py file, and added to the Azure SQL database, then displayed to the user.

![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/ee30f32b-4e94-4cb8-93f0-2c17074a8701)

![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/14a01d48-df86-4bd8-b196-7342d19f07d5)


The 'delete' function can be used to remove an item from the todolist, deleting it from the database, and the 'mark as done' function can be used to cross out a todolist item.
	
![image](https://github.com/romhaki/Migrating-a-Flask-Web-App-To-Azure/assets/136436650/c48bf483-30a7-43c9-8024-67d1dd5cd370)

	

 </details>
    
  
    
