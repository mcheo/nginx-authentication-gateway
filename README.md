# nginx-authentication-gateway

1 dev team mentioned their microservices need to call outside cluster API services. They can influence how the outside cluster API implements the authentication, in this case JWT validation. But they find it challenging to do key management for various microservices. Should they put the JWT token as k8s secret for these microservices? How about renewing the expired token? Revoking the token? and etc

This repo demonstates how to leverage NGINX Plus as the gateway to authenticate with outside cluster API services.

## Prerequisite: 
NGINX Plus [You may request for a NGINX Plus [Trial License](https://www.nginx.com/free-trial-request/) and build the [NGINX Plus Docker Image](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-docker/)]

- Install docker and docker-compose in your machine
- Build NGINX Plus docker image and have it in your machine (You do not need to push to external repo)


## Getting Started
To start this demo:
```
#Clone the repo
git clone https://github.com/mcheo-nginx/nginx-authentication-gateway

#To step up the stack
docker-compose -f docker-compose.yml up -d 

#To delete the demo stack after testing
docker-compose -f docker-compose.yml down
```

We leverage docker-compose to spin up 3 components:
- NGINX-1 - As the proxy where microservices will call, it will inject relevant Authorization bearer token and send to upstream servers
- NGINX-2 - As the sample proxy for the backend API endpoints, implement JWT validation for any API request
- HTTPBIN - As API endpiont servers


In NGINX-1, we leverage NGINX Plus [Key-Value store](http://nginx.org/en/docs/http/ngx_http_keyval_module.html) feature to dynamically Add/Remove/Update in memory table. We will centrally store our JWT tokens in this table. In this simple demo, we will use request URI as "Key" to locate the relevant token to send to backend servers. NGINX supports various parameters as "Key".

I have created 2 JWK secret "apisecret_1" and "apisecret_2" that NGINX-2 will use to validate the token, we will use these to generate valid JWT token. Once we have the valid tokens, we will add them into our Key-Value table.

Sample JWT token:
```
Use apisecret_1 as secret
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InRlc3RpbmcifQ.g6Qc7JFdShojUrYthuf3sl57SrygzHba7qIBFnpx_Vs

Use apisecret_2 as secret 
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InRlc3RpbmcyIn0.D6JNQBzNoxzolnexo2N_kJH8prYZOv74x7BeqCLYNo8
````

I have defined Key-Value zone as jwtzone in the conf file. Sample commands to update NGINX Plus Key-Value store, 
```
#To list table
curl  http://localhost:8080/api/7/http/keyvals/jwtzone

#To add record into the table
curl -X POST -d '{"/get":"<sample value>"}'  http://localhost:8080/api/7/http/keyvals/jwtzone

#To remove record from the table
curl -X PATCH -d '{"/get":null}' -s 'http://localhost:8080/api/7/http/keyvals/jwtzone'
```

Step 1:
```
#In your workstation execute these commands. Both requests will fail as there are no valid JWT token send by NGINX-1 to NGINX-2
curl http://localhost:81/get
curl http://localhost:81/headers
```
Step 2:
```
#Add valid JWT token for /get URI
curl -X POST -d '{"/get":"Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InRlc3RpbmcifQ.g6Qc7JFdShojUrYthuf3sl57SrygzHba7qIBFnpx_Vs"}'  http://localhost:8080/api/7/http/keyvals/jwtzone

#This request will goes through as there is valid token
curl http://localhost:81/get

#This request will not goes through as there is no valid token
curl http://localhost:81/headers

```

Step 3:
```
#Add valid totken for /headers URI
curl -X POST -d '{"/headers":"Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6InRlc3RpbmcyIn0.D6JNQBzNoxzolnexo2N_kJH8prYZOv74x7BeqCLYNo8"}'  http://localhost:8080/api/7/http/keyvals/jwtzone

#This request will goes through as there is valid token
curl http://localhost:81/headers

```
Step 4:
Once you remove the token from the Key-Value table, these request will fails.



## Other possibilities
- We use NGINX API to Add/Remove/Upate Key-Value store, dev team can integrate into their pipeline to automate the key management operation
- Currently we use request URI as key to locate the relevant JWT token, we can consider using header, parameters or other variables. This can be used to impose certain check on microservices requests to NGINX-1
- We can create different Key-Value zone for different requirements



## References:
- [NGINX Official Docs](https://docs.nginx.com/)
- [NGINX Modules Documentation](http://nginx.org/en/docs/)
