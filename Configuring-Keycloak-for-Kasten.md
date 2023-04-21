In this article, we discuss the steps for installing, configuring and running Keycloak on Linux server that can be used as an Identity Provider for Kasten RBAC testing. The instructions are based on Keycloak running on centos9.

### 1. Get Keycloak Pre-requisites

SSH into the linux host where keycloak will be installed. Check if Java is installed using `java -version` command. If not, install using `yum install -y java`

### 2. Download Keycloak

```
cd /opt
sudo wget https://github.com/keycloak/keycloak/releases/download/21.0.2/keycloak-21.0.2.tar.gz
sudo tar -xvzf keycloak-21.0.2.tar.gz
sudo mv keycloak-21.0.2 keycloak
```

Add a firewall rule to allow port 8083 for running Keycloak UI

```
firewall-cmd --permanent --zone=public --add-port=8083/tcp
firewall-cmd --reload
```
 
When running Keycloak for the first time, you need to set the Keycloak admin user and password. This can be done defining two environment variables KEYCLOAK_ADMIN and KEYCLOAK_ADMIN_PASSWORD
 
```
export KEYCLOAK_ADMIN=kcadmin
export KEYCLOAK_ADMIN_PASSWORD=kcadmin
```

Start Keycloak
```
/opt/keycloak/bin/kc.sh start-dev --http-port 8083
```

### 3. Log into Keycloak dashboard

Keycloak dashboard can be accessed using the below URL and the credentials defined in Step 2

Login URL : http://<centos9_hostname>:8083/admin/master/console/

Login Admin Credentials :  kcadmin / kcadmin


### 4. Configuring Keycloak for using with Kasten

Log into Keycloak dashboard using admin credentials  

#### 4.1. Create a new realm

Keycloak creates a default master realm. For kasten, let’s create a new realm.

Click the dropdown next to master realm -> Create Realm -> Enter the realm name and click Create.  In this example, i am using a realm name k10-test-realm

<img width="270" alt="image" src="https://user-images.githubusercontent.com/2148411/233654011-7f42b28e-9a43-4b7a-bfb1-5f9697dca225.png">

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233653912-4b877f4f-c203-4738-b5a3-b45dd158b4a1.png">

#### 4.2. Create Test Groups

Select Groups on the Left menu

Click Create Group and provide group name. I have created three groups (idp-k10admingrp, idp-k10devgrp1, idp-devgrp2)

<img width="280" alt="image" src="https://user-images.githubusercontent.com/2148411/233653665-76fe7fe3-9506-4524-90ea-598d4a23075a.png">

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233653773-992ba9b5-174d-4ab1-947b-8390f28f1d18.png">

#### 4.3. Create Test Users

Select Users on the Left menu

Click Add user -> Provide Username, Email, First and Last name

Enable `Email Verified`

Click Join Groups and select the group to which the user belongs. I have selected `idp-devusr1@k10.lab` to be part of `idp-devgrp1`

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233654780-5242f1de-6cb2-435b-84ee-b89c214dd589.png">

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233654944-74d873fe-9d73-4f59-a1da-ef5735314dae.png">

Select the user that was created to open its details and click Credentials tab. By default, there will be no user credentials. Create one by clicking Set password

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233655804-b3bba18c-ee64-4411-b6e1-096ae8f3a9e6.png">

Provide the password for the user and disable Temporary. Click Save

<img width="546" alt="image" src="https://user-images.githubusercontent.com/2148411/233655530-56243da7-b985-4609-8da7-2e3c1b525264.png">


#### 4.4. Create a new Client for Kasten

Clients are entities that can request Keycloak to authenticate a user. Most often, clients are applications and services that want to use Keycloak to secure themselves and provide a single sign-on solution - this is how we will use Keycloak for the K10 installation.

Select the realm k10-test-realm that was created for Kasten to be the current realm.

In the left menu, click clients --> Create Client

Under General Settings – Specify the Client ID, Name, Description and Turn on Always display in UI

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233657621-1a473891-a3da-4031-b191-e15de13a63f9.png">

Click Next. Under Capability Config – Enable Client Authentication and Implicit flow. Click Next

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233657824-bcd59ef7-554a-4132-a2b6-d61ad6cb4931.png">

Under Login Settings

Add the following two URLs to `Valid Redirect URIs`

```
http://127.0.0.1:8080/*
https://oidcdebugger.com/debug
```

Specify Web Origins as `/*`

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233658507-51c22737-22d1-477c-a6dd-b5818c864716.png">

Click Save to create the client

#### 4.5. Kasten Client configuration

Select Clients from left menu  -> Click Kasten -> Client Scopes -> Kasten-dedicated

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233659558-de11cc57-f722-4065-bc88-15dae79fb1b0.png">

By default, there will be no mappers created. Click Configure a new mapper.

Select `Group Membership` under the Mapper Name

Enter the name of the mapper as `groupmapper`

Enter the Token Claim Name as `groups`

Enable `Add to ID Token`, `Add to access Token`, `Add to userinfo`

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233660008-ae96f2b2-b2f6-422c-a9bc-90264aa2505b.png">

Click Save

#### 4.6. Test Keycloak Authentication (Outside of Kasten)

In a new browser tab, open https://oidcdebugger.com/

Under Authorize URI, provide the below URL. Change the hostname and port number according to your environment

`http://<keycloak_host>:8083/realms/k10-test-realm/protocol/openid-connect/auth`

Under Client ID, specify `Kasten` to match your Keycloak client name.

Under Response Type ->  Select `id_token` (unselect the other options if enabled)

Under Response Mode -> Select `form_post`

Click Send Request

You should now see KeyCloak Login screen.

Enter the login credentials of a user. (In my test, I provided the credentials for idp-devusr1@k10.lab user)

The response received should be similar the one below. In the Payload -> Groups, I can see the user `idp-devusr1` belongs to group `idp-devgrp1`

<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233667884-b0276897-4e1a-4041-8be2-766a0758c6d0.png">


#### 4.7. Installing Kasten with the OIDC auth to Keycloak

To install kasten using the Keycloak identity provider, we need to provide additional information for k10 OIDC client to authenticate with Keycloak. Create a values.yaml file similar the one below. 

```
cat > values.yaml << EOF
auth:
  oidcAuth:
    enabled: true
    clientID: Kasten
    clientSecret: <xxxxxxxxxxxxxx>  
    providerURL: http://<keycloak_host>:8083/realms/k10-test-realm
    redirectURL: http://<kasten_host>:<kasten_port> 
    scopes: "openid"
    usernameClaim: preferred_username
    groupClaim: groups
    usernamePrefix: "-"
    prompt: login
EOF
```

Provide new values for clientSecret, providerURL and redirectURL according to your environment. 
 
Use the following steps to obtain clientSecret:
 
Open the Keycloak Admin Console and login using admin credentials
 
Select Clients > Kasten and click the Credentials tab.
 
Copy the Secret displayed in the UI.
 
<img width="800" alt="image" src="https://user-images.githubusercontent.com/2148411/233669935-adf7b8be-9ae2-4515-ab33-a659b66438e1.png">

Under Settings tab --> Access Settings --> Valid redirect URIs, add the following URL. Replace Kasten_host and Kasten_port according to your environment.

`http://<kasten_host>:<kasten_port>/k10/auth-svc/v0/oidc/redirect`

Install Kasten using the helm command
 
`helm install k10 kasten/k10 --namespace=kasten-io -f values.yaml`

Verify all pods are running fine

`Watch -n2 kubectl get pods -n kasten-io`
