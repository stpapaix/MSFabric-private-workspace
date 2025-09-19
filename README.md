# Helper to demonstrate MS Fabric Private Workspace


## Objectives

Secure the Inbound access of a Microsoft Fabric Workspace with Azure Private Link and Azure Networking private endpoints.

Link to Microsoft documentation :
[https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-overview](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-overview)

Verify the following prerequisites
[https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#prerequisites](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#prerequisites)

## Configure MS Fabric Tenant

To can configure inbound private link access protection in workspace settings, you have to authorize this action in Fabric Tenant Admin Portal.
In "Tenant Settings", look at "Configure workspace-level inbound network rules" (in Advanced Networking section)
![alt text](assets/workspace-inbound-network-rules.png)

This Objective of this post is to protect a MS Fabric Workspace from public access and authorize only access through private link like represented in the following schema:

![](assets/20250919_220122_Workspace-private-link-for-Fabric-scaled-3.png)

## Create Private Link service for Fabric for Workspace

You have to choose the private link you want to secure.

You need to capture, **your tenant ID**, and your **workspace ID**.

### Tenant ID

Method in Fabric Portal:

- Open the Microsoft Fabric portal
- Click the question mark icon (?) in the top-right corner.
- Select “About Fabric”.
- Look for the Tenant URL — your Tenant ID is the value of the`ctid` parameter in that URL.

Simpler method with Azure Portal, go to Entra ID.

### Workspace ID

From your Fabric portal URL

"https://app.fabric.microsoft.com/groups/`2983gbr-51e6-4687-b036-e5f4b42b3e3c`/list?experience=fabric-developer)

Note : the Worspace ID is split into 5 groups separated by hyphens, following the standard UUID/GUID representation. Web app URLs use the standard GUID notation, so the portal shows it with hyphens for readability and consistency with API responses. (8-4-4-4-12). DNS hostnames can’t contain hyphens in certain positions that would break the label structure, so Microsoft strips them out when embedding the GUID into the FQDN for the data‑plane endpoint.
Your Workspace ID will be : `2983gbr51e64687b036e5f4b42b3e3c`

Now it's time to create your private Endpoint for your MS Fabric Workspace

Look at the MS documentation for that:

[Set up and use workspace-level private links - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#step-3-create-a-virtual-network)

You created a `microsoft.fabric/privatelinkservicesforfabric`

To check go with Azure portal on the Ressource "Azure Resource Graph Explore", and execute the following query:

```
resources 
| where type =~ "Microsoft.Fabric/privateLinkServicesForFabric" 
| project id, name, type, location
```

![](assets/20250919_184641_privatelinkservicesforfabric.png)

Now it's time to create the infrastructure to implement your private endpoint to access to you Worksapce through the Fabric Private Link

[Create a Virtual Network](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#step-3-create-a-virtual-network)

[Create a VM](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#step-4-create-a-virtual-machine)

[Create a Private Endpoint](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-set-up#step-5-create-a-private-endpoint)

Now you will have something like this schema

![](assets/20250919_193335_Workspace-private-link-for-Fabric-scaled-1.png)

* A Private Link connected to your MS Fabric Workspace.
* A Private Endpoint in your VNet linked to the Private Link.

Warning: If your VM has a public IP, verify the VNet Link exist for the `Private DNS` created with during private link setup.

In azure portal go to "Private DNS zones

![](assets/20250919_194849_privatednszone-1.png)

And look at the Private DNS Zone to check if there is a link for your VNet

![](assets/20250919_195234_privatednszone-2.png)

You can also look at the Recordsets to see all record defined for your Workspace services (like blob, api, onelake...) corresponding with Azure private IP.

## Test private connectivity

Now we have all the elements to establish a private (non public) connexion from the VM to your Microsoft Fabric Workspace.

We are going to verify:

* Start and connect to your VM,
* Start a powershell Command Session
* nslookup to verify the DNS return a private adress for your API ({workspaceID}.z{xy}.w.api.fabric.microsoft.com)

![](assets/20250919_200149_nslookup.png)

In this case the result is OK, 172.16.0.5 is a private IP.

* Test-NetConnection powershell command to test a TCP access to the API.

![](assets/20250919_200422_Test-NetConnection.png)

Result OK private IP.

## Deny Inbound Access to the Workspace

Now it is time to deny the public access to the Workspace.

![](assets/20250919_202105_Workspace-private-link-for-Fabric-scaled-2.png)

To modify or deny the Workqpace public access rules use the [Workspaces - Set Network Communication Policy API](https://learn.microsoft.com/en-us/rest/api/fabric/core/workspaces/set-network-communication-policy)

There is differents ways to call the API, in this helper book we are going to use pyspark code to be able to execute the pyspark code from a Note Bok inside MS Fabric (Obviously from a Workspace with publi access).

Pyspark code to verify the Workspace Public Access Rules:

```python
# Verification de l'etat des authorisation Inbound et Outbound d'un Workspace
from notebookutils import mssparkutils
import requests

WID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Recuperation du token
token = mssparkutils.credentials.getToken("https://api.fabric.microsoft.com")

# Contruction du header
h = {"Authorization": f"Bearer {token}", "Content-Type":"application/json"}

# Test GET (reponse 200 attendue)
get_request = requests.get(f"https://api.fabric.microsoft.com/v1/workspaces/{WID}/networking/communicationPolicy", headers=h)
print(get_request.status_code, get_request.text)
```

200 {"inbound":{"publicAccessRules":{"defaultAction":"Allow"}},"outbound":{"publicAccessRules":{"defaultAction":"Allow"}}}

Now the Pyspark code to modify the Workspace Public Access Rules:

```python
# Modification des authorisation Inbound et Outbound d'un Workspace
from notebookutils import mssparkutils
import requests, json, time

WID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx""

# Valeur cible Inbound defaultvalue
target_inbound_value = "Deny"

# Recuperation du token Fabric API
token = mssparkutils.credentials.getToken("https://api.fabric.microsoft.com")

# Contruction du header
h = {"Authorization": f"Bearer {token}", "Content-Type":"application/json"}

# Recupèration de la policy actuelle
current_policy = requests.get(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WID}/networking/communicationPolicy", 
    headers=h
    ).json()

print("Policy actuelle: ", current_policy)

# Contruction de la policy avec Inbound = Deny
new_policy = current_policy
new_policy.setdefault("inbound", {}).setdefault("publicAccessRules", {})["defaultAction"] = target_inbound_value

# Application de la policy 
put_request = requests.put(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WID}/networking/communicationPolicy", 
    headers=h,
    data=json.dumps(new_policy)
)
print("PUT Status", put_request.status_code, put_request.text)

# Boucle pour attendre la maj
for _ in range(4):
    time.sleep(1)
    test_result = requests.get(
        f"https://api.fabric.microsoft.com/v1/workspaces/{WID}/networking/communicationPolicy",
         headers=h
    ).json()
    print("defaultAction:", test_result)
    if test_result.get("inbound", {}).get("publicAccessRules", {}).get("defaultAction") == target_inbound_value:
        break

```

Policy actuelle:  {**'inbound': {'publicAccessRules': {'defaultAction': 'Allow'}**}, 'outbound': {'publicAccessRules': {'defaultAction': 'Allow'}}}

PUT Status 200
defaultAction: {**'inbound': {'publicAccessRules': {'defaultAction': 'Deny'}**}, 'outbound': {'publicAccessRules': {'defaultAction': 'Allow'}}}
