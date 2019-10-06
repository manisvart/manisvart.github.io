---
layout: post
title: Sharepoint to Neo4j
categories: [Projects, Coding, Python, Graph]
---
At [work](https://www.arbetsformedlingen.se/Globalmeny/Other-languages/Languages/English-engelska.html){:target="_blank"} we have a homebuilt tool called "AFFIRM" which is a javascript frontend using [d3js](https://d3js.org){:target="_blank"} to visualize data from [Neo4j](https://neo4j.com){:target="_blank"}. The data in Neo4j comes from a SharePoint site and this automates the data transfer from Sharepoint to Neo4j.

AFFIRM/Neo4j runs on a Linux machine and SharePoint runs on a Windows machine (duh!). We'll use Python to extract the data.

I've compiled the result of my research into this subject from multiple sources, as there were no readily available comprehensive information about how to do this, and I share the result with you.

## SharePoint lists

The data in Neo4j comes from a SharePoint site with a couple of lists. The lists use the SharePoint functionality "lookup column with an enforced relationship" which gives us referential integrity between the lists.

The lists:

- **nodes** Contains the nodes with metadata.
  - nodeTitle  
    The title of the node
  - nodeType  
    The type of the node such as "system", "knowledge", "capability", …
  - fromYear
  - toYear
  - portfolio  
    The product portfolio that this node belongs to.
  - more  
    A link to more information about the node. Opened in a new window.
- **relations** Contains the relations between the nodes.
  - fromNode
  - relType  
    The type of relation such as "depends on", "contributes to", "data transfer", …
  - relData  
    A short note about the relation.
  - toNode
  - fromYear
  - toYear
- **portfolio** A lookup list for the product portfolios.
  - portfolio
- **nodeType** A lookup list for the node types.
  - nodeType
- **relType** A lookup list for the relation types.
  - relType

*Please note that if you create a field (column) in SharePoint and later rename it, you'll still see, and must use, the original name in the API.*

## SharePoint API

SharePoint has a REST API documented [here](https://docs.microsoft.com/en-us/sharepoint/dev/sp-add-ins/get-to-know-the-sharepoint-rest-service){:target="_blank"} that we'll use to get information from the lists. We'll use the endpoint:

```html
http://server/site/_api/lists/getbytitle('listname')/items
```

…to get the records from the lists. Each call will return a limited number of records, so for long lists you'll get a `__next` parameter that'll get you the next batch of records. See the code for more details.

To access SharePoint we'll use a separate "service account". This account has to be added to the SharePoint site as a user to be able to access the lists.

So how do you authenticate to your SharePoint site? That depends on how the SharePoint IIS is configured. If you don't have access to the machine or anyone to answer your questions you can use the following method at the Python console on the Linux machine after you have installed `requests`:

```bash
sudo pip install requests
```

Start Python and:

```python
import requests

r = requests.get("http://server/site/_api/lists/getbytitle('listname')/items", verify=false)
r.status_code
r.headers['www-authenticate']
```

The `verify=false` tells the `requests` library to not verify the SharePoint servers certificate. It's common in organizations to use internal certificate issuers and until your Linux server is a member of that certificate structure you'll need `verify=false` to connect.

You'll probably get `401` as a result of `r.status_code`. In my case `r.headers['www-authenticate']` gave `'Negotiate, NTLM'`. So we need to use NTLM to authenticate.

## Python

To install support for NTLM authentication we need to install two packages on the Linux machine:

```bash
sudo pip install ntlm-auth
sudo pip install requests_ntlm
```



## Neo4j

To add the data to the graph database we'll use the `cypher-shell` [command](https://neo4j.com/docs/operations-manual/current/tools/cypher-shell/){:target="_blank"}.

 

## The Python script

Since we're in Sweden and have all those funny accented characters like åäöÅÄÖ we'll use UTF-8 encoding.

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# ---------------------------------------------------------------------------
# @author Carl Nordin (manisvart)
# @copyright (C) 2018,Carl Nordin (manisvart)
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# ---------------------------------------------------------------------------

import sys

reload(sys)
sys.setdefaultencoding('utf8')

import requests
from requests_ntlm import HttpNtlmAuth

import json

# SharePoint authentication information
sp_account  = "domain\\sp_account"
sp_password = "sp_password_åäö"
sp_site     = "https://sharepoint.yourorganization.com/sites/affirm"

# SharePoint HTTP
sp_headers = {
	'accept' : 'application/json;odata=verbose',
	'content-type' : 'application/json;odata=verbose',
	'odata' : 'verbose',
	'X-RequestForceAuthentication' : 'true'
}

# Neo4j authentication
n4j_account  = "neo4j_account"
n4j_password = "neo4j_password"

n4j_export_file = "./cypher_shell_import.text"

# Get information from the API, extract the fields we're interested in from the 
# records and store in a dictionary.
#
# NOTE! We're using "verify = False" here to avoid certificate check. Your Linux
# machine should be a part of your certificate structure. 
def get_sp_items(session, suburl, fields, id_field = 'ID'):
	result = {}

	url = sp_site + suburl

	while url != "":
		r = session.get(url, verify = False, headers = sp_headers)
		j = json.loads(r.text)

		# The result is a dictionary in two levels: {'d': 'results': ...}
		res = j['d']['results']

		# Save the specified fields
		for item in res:
			f = {}

			# The specified "ID" field *must* exist
			id = item[id_field]

			for subitem in item:
				# Uncomment the following line to see the column names and values
				#print "--", subitem, "--", item[subitem] ,"--"

				try:
					i = fields.index(subitem)
				except ValueError:
					i = -1

				if i != -1:
					f[subitem] = item[subitem]

			result[id] = f

		# Is there more data available?
		try:
			url = j['d']['__next']
		except KeyError:
			url = ""

	return result


# Start
session = requests.Session()
session.auth = HttpNtlmAuth(sp_account, sp_password)

# Get the node types
node_type_fields = ['Title']
node_types = get_sp_items(session, "/_api/lists/getbytitle('nodeType')/items", node_type_fields, 'ID')

# Get the relation types
rel_type_fields = ['Title']
rel_types = get_sp_items(session, "/_api/lists/getbytitle('relType')/items", rel_type_fields, 'ID')

# Get the product portfolios
portfolio_fields = ['Title']
portfolio = get_sp_items(session, "/_api/lists/getbytitle('portfolio')/items", portfolio_fields, 'ID')

# Get the nodes
node_fields = ['Title', "nodeTypeId", "toYear", "fromYear", "more", "portfolioId"]
nodes = get_sp_items(session, "/_api/lists/getbytitle('nodes')/items", node_fields, 'ID')

# Get the relations
relation_fields = ['relData', 'fromNode', 'toNode', 'relTypeId', 'toYear', 'fromYear', 'more']
relations = get_sp_items(session, "/_api/lists/getbytitle('relations')/items", relation_fields, 'ID')

# Create an import file for Neo4j's cypher-shell
f = open(n4j_export_file, "w")

# Delete all previous nodes and relations
f.write("MATCH (n) DETACH DELETE n;" + '\n')

# Create Neo4j nodes
for node_id in nodes:
	n = "MERGE("
	n += ":" + node_types[nodes[node_id]['nodeTypeId']]['Title']
	n += "{"
	n += "affirmID:'i" + str(node_id) + "'"
	n += ",title:'" + nodes[node_id]['Title'] + "'"
	if nodes[node_id]['fromYear'] != None:
		n += ",fromYear:" + str(nodes[node_id]['fromYear'])
	if nodes[node_id]['toYear'] != None:
		n += ",toYear:" + str(nodes[node_id]['toYear'])
	if nodes[node_id]['portfolioId'] != None:
		n += ",portfolio:'" + portfolio[nodes[node_id]['portfolioId']]['Title'] + "'"
	if nodes[node_id]['more'] != None:
		n += ",more:'" + str(nodes[node_id]['more']['Url']) + "'"
	n += "});"
    
	f.write(n + '\n')

# Create neo4j relations
for relation_id in relations:
	r = "MATCH(a),(b) WHERE"
	r += " a.affirmID='i" + str(relations[relation_id]['fromNode']) + "'"
	r += " AND b.affirmID='i" + str(relations[relation_id]['toNode']) + "'"
	r += " MERGE (a)-[r:" + rel_types[relations[relation_id]['relTypeId']]['Title']
	r += "{"
	if relations[relation_id]['relData'] != None:
		r += "relData:'" + relations[relation_id]['relData'] + "'"
	else:
		r += "relData:''"
	if relations[relation_id]['fromYear'] != None:
		r += ",fromYear:" + str(relations[relation_id]['fromYear'])
	if relations[relation_id]['toYear'] != None:
		r += ",toYear:" + str(relations[relation_id]['toYear'])
	if relations[relation_id]['more'] != None:
		r += ",more:'" + relations[relation_id]['more'] + "'"
	r += "}]->(b) RETURN r;"

	f.write(r + '\n')

f.close()

# And do the actual import (redirect chyper-shell's output to /dev/null)
import os
os.system("cat " + n4j_export_file + " | cypher-shell -u " + n4j_account + " -p " + n4j_password + " > /dev/null")

exit(0)
```



And schedule it using `crontab`:

```bash
0 1 * * * python sp_neo4j.py 2> /dev/null
```


