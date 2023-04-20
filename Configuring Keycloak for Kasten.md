In this article, we discuss the steps for installing, configuring and running Keycloak on Linux server that can be used as an Identity Provider for Kasten RBAC testing. The instructions are based on Keycloak running on centos9.

### Get Keycloak Pre-requisites

SSH into the linux host where keycloak will be installed. Check if Java is installed using `java -version` command. If not, install using `yum install -y java`

### Download Keycloak

```
cd /opt
sudo wget https://github.com/keycloak/keycloak/releases/download/21.0.2/keycloak-21.0.2.tar.gz
sudo tar -xvzf keycloak-21.0.2.tar.gz
sudo mv keycloak-21.0.2 keycloak
```

Add firewall rules to allow port 8083 used for running Keycloak UI

```
firewall-cmd --permanent --zone=public --add-port=8083/tcp
firewall-cmd --reload
```
 

export KEYCLOAK_ADMIN=kcadmin
export KEYCLOAK_ADMIN_PASSWORD=kcadmin
/opt/keycloak/bin/kc.sh start-dev --http-port 8083

 

Keycloak Admin URL and credentials

Login URL : http://wasp-centos9.sbmlabs.net:8083/admin/master/console/

Login Admin Credentials :  kcadmin / kcadmin

Configuring Keycloak for using with Kasten



Log into Keycloak Administration UI

http://wasp-centos9.sbmlabs.net:8083/admin/master/console/

kcadmin / kcadmin

  

Create a new realm

Keycloak creates a default master realm. For our purposes, let’s create a new realm.

Click the dropdown next to master realm -> Create Realm → Enter the realm name and click Create. In this example, i am using a realm name k10-test-realm

![image](https://user-images.githubusercontent.com/2148411/233468091-68659681-5526-462d-98b5-56e0de96b423.png)

![image](https://user-images.githubusercontent.com/2148411/233468335-58dbf023-385c-4cd4-b1fa-7d7e8a7fc44e.png)

Create Test Users and Groups

 

Select Groups on the Left menu

Click Create Group and provide group name. In my environment, I have created three groups.

![image](https://user-images.githubusercontent.com/2148411/233468468-0362df06-d4f2-410a-b460-3501c600174d.png)



Select Users on the Left menu

Click Add user

Provide user name, First and Last name

Enable `Email Verified`

Click Join Groups and select the group to which the user belongs.

![image](https://user-images.githubusercontent.com/2148411/233468570-c4fc3002-dd78-4a69-8d40-57def60c7049.png)


Select the user to open its details and click Credentials tab. 

By default, there will be no user credentials. Create one by clicking Set password

![image](https://user-images.githubusercontent.com/2148411/233468667-f02ea948-919c-4f25-8620-5e29d4042838.png)



Provide the password for the user and turn of Temporary. Click Save

![image](https://user-images.githubusercontent.com/2148411/233468767-b722737c-75f2-4ba4-9c7e-b3c4bee1c016.png)



Create a new Client

Select the realm `k10-test-realm` that was just created to be the current realm.

Select `Clients` at the left dropdown and create a new client.

Under General Settings – Specify the Client ID, Name, Description and Turn on Always display in UI

![image](https://user-images.githubusercontent.com/2148411/233468819-8641f4bf-7b7c-48e1-afaf-5ad90eee46bb.png)


Under Capability Config – Enable Client Authentication and Implicit flow. Click Next


![image](https://user-images.githubusercontent.com/2148411/233468924-ad1a38c8-ef61-4b14-923f-2bae5f668b74.png)


Under Login Settings

Add the following Valid Redirect URIs

http://127.0.0.1:8080/*

http://kind-k10-centos9.sbmlabs.net:32200/k10/auth-svc/v0/oidc/redirect

https://oidcdebugger.com/debug 

Add Web Origins as `/*`

Click Save to create the client

![image](https://user-images.githubusercontent.com/2148411/233469014-d400ddcc-a459-48d2-8295-e9dede84a7c4.png)

### Additional Kasten Client configuration

 
 
Select Clients -> Kasten -> Client Scopes -> Kasten-dedicated

![image](https://user-images.githubusercontent.com/2148411/233469099-2d32f1e7-7eba-4f7d-ae7b-db497a3c393a.png)


Select Group Membership under the Mapper Name

Enter the name of the mapper as ` groupmapper`

Enter the Token Claim Name as `groups`

Enable Add to ID Token, Add to access Token, Add to userinfo

Click Save

Test Keycloak Authentication (Outside of Kasten)

In a new browser tab, open https://oidcdebugger.com/ .

Under Authorize URI, provide. Change the hostname and port number according to your environment

http://wasp-centos9.sbmlabs.net:8083/realms/k10-test-realm/protocol/openid-connect/auth

Under Client ID, specify Kasten to match your Keycloak configuration.

Under Response Type ->  Select id_token only

Under Response Mode -> Select form_post

Click Send Request

You should now see KeyCloak Login screen.

Enter the login credentials. (in my case, I provided the credentials for idp-devusr1@k10.lab user)

The response received should be similar the one below. In the Payload -> Groups, I can see the user idp-devusr1 belongs to idp-devgrp1.

![image](https://user-images.githubusercontent.com/2148411/233469159-9ce1bb90-f7cc-4d57-9fc6-933571fea908.png)


Installing Kasten with the OIDC auth to Keycloak



cat > values.yaml << EOF
auth:
  oidcAuth:
    enabled: true
    clientID: Kasten
    clientSecret: GfEnVDuxUscjZ5FylMJHxwS5UHGCGzTi  
    providerURL: http://wasp-centos9.sbmlabs.net:8083/realms/k10-test-realm
    redirectURL: http://kind-k10-centos9.sbmlabs.net:32200 
    scopes: "openid"
    usernameClaim: preferred_username
    groupClaim: groups
    usernamePrefix: "-"
    prompt: login
EOF


helm install k10 kasten/k10 --namespace=kasten-io -f values.yaml

Verify all pods are running fine

