# Start vault and rabbitmq containers

The first thing we need to do it start vault and RabbitMQ, to do this we are just going to use a containerised versions of the apps running in dev mode.

> :warning: **Dont do this in PROD**: This is a quick lab to walk though how vault works, non of this is prod ready

If on the devops playground lab: `cd` in to the lab root

`cd workdir/Hands-on-with-Hashicorp-Vault`

To start the containers run docker compose:
`docker-compose up -d`

With in the `docker-compose.yml` file we start a vault and rabbitMQ server.

This is the command we use to run vault:
`command: ["server", "--dev", '-dev-root-token-id=root']`
It starts vault in dev mode, with `root` as the root token.

## Go login to vault

Now that vault is up time to get familiar with the UI, even though we can (and will) be doing everything with API calls the UI is a nice way of double checking everything works.

In your browser go to `http://<your panda>.devopsplayground.org:8200/` and in the screen type `root` as the token

![login](images/login.png)

Have a look around the other parts of vault from the nav bar.
![nav](images/nav.png)

## Set up auth method - userpass

We have seen the UI but its not got a lot going on at the moment, lets set up an auth method so our users can log in.

### Create auth method

Run the following curl request. This will make a auth method of type userpass at the path `/userpass`

``` bash
curl --request POST  http://127.0.0.1:8200/v1/sys/auth/userpass \
    --header "X-Vault-Token: root" \
    --data-raw '{
  "type": "userpass"
}'
```

### Create policy

> :exclamation: depending on the auth method used it refers to how you are login diffrently. As we are using userpass I will be talking about user, but could be replces with role if talking about a diffrent auth method

Vault uses policies to let user do things. Before we create a user let create the policy that will let it read the rabbitMQ login details. This will create a policy called `rabbitmq` that has access to read the `rabbitmq/creds/chat` path

``` bash
curl --request PUT http://127.0.0.1:8200/v1/sys/policies/acl/rabbitmq \
    --header "X-Vault-Token: root" \
    --data-raw '{
  "policy": "path \"rabbitmq/creds/chat\" {\n capabilities = [\"read\"]\n}"
}'
```

### Create entity

While we could just create a user, we want to alow our app to be easly extended in the future so we are going to create a identity, this will allow us to change (or add) the auth method in the future with out having to change the core functionality.

The below curl request makes a identity called panda with the policy we attached earlyer. If you wanted to make another user you would just have to rerun the folowing 3 curl commands changeing where you see panda to something else.

This comand make an entity with the name panda and attaches the rabbitmq policy we made earlyer

``` bash
curl --request POST http://127.0.0.1:8200/v1/identity/entity \
   --header "X-Vault-Token: root" \
   --data-raw '{
  "name": "panda",
  "policies": ["rabbitmq"]
}'
```

### Create user

Now we have all the ground work set up, time to make the user we can log in with.

Below is the curl command to make a user with the name panda and the password pass.

This command makes a user called `panda` and sets its password to `pass`

``` bash
curl --request POST http://127.0.0.1:8200/v1/auth/userpass/users/panda \
   --header "X-Vault-Token: root" \
   --data-raw '{
    "password": "pass"
}'
```

### Link user to identity

If you now try and log in with the created user, you will find it cant do anything. This is because the identity has the permissions and we havent linked them together yet.  

In the below command you will need to get the canonical_id from the user you just created, and the Mount_accessor we created at the start, if you scrol up you might be able to see them or you can find from the UI:

From the UI go to the Access from the nav bar.

UserID:
![UserID](images/userID.png)

If you would rather get this infomation from the command line:

`curl http://127.0.0.1:8200/v1/identity/entity/name/panda --header "X-Vault-Token: root" | jq ".data.id"`

Auth method Accessor:
![Auth method](images/mount.png)

If you would rather get this infomation from the command line:

`curl http://127.0.0.1:8200/v1/sys/auth/userpass --header "X-Vault-Token: root" | jq ".accessor"`

``` bash
curl --request POST http://127.0.0.1:8200/v1/identity/entity-alias \
   --header "X-Vault-Token: root" \
   --data-raw '{
  "name": "panda",
  "canonical_id": "<User id>",
  "mount_accessor": "<Auth method accessor>"
}'
```

## Add code to go to get the auth token

Now that we have created a user, let add the code to allow us to log in.

Open up `./main.go`

Replace the import block at the top with

``` go
  import(
    "log"
    "net/http"
    "fmt"
    "net/url"
    "io"
    "encoding/json"
)
```

Then under the comment `//code to get go token` add the following

``` go
  vaulturl := "http://localhost:8200" //the vault server url
  username := "panda" //The username for the userpass auth method
  resp, err := http.PostForm(
  fmt.Sprintf("%s/v1/auth/userpass/login/%s",vaulturl, username), //builds the url for the login endpoint
  url.Values{"password":{"pass"}}) //passess in the password we set earlyer (could be grabed from user input)
  failOnError(err, "failed to log in to vault")
  defer resp.Body.Close()
  body, err := io.ReadAll(resp.Body) //reads the body of the responce
  failOnError(err, "failed to read body")
  fmt.Print(string(body))
  var up userpass
  err = json.Unmarshal(body,&up) //converts the body in to a go struct so we can get the token
  failOnError(err, "failed to Unmarshal body")
  fmt.Println(up)
```

## Create secrets engine

Now we can login with our app, its not very useful unless we enable a secrets engine. We are going to use rabbitMQ as our secrets engine, this will genirate a rabbitMQ username and password when you hit the `http://127.0.0.1:8200/v1/rabbitmq/creds/<role>`. While this might not seem useful using a username and password to genirate a username and password. You could enable another authmethod to let a user athenticate with rabbitMQ by just running the applicaiton on the correct system.

### Create secret mount

The first step to use a secret engine in vault is to enable the engine mount. Below is the curl command to enable the rabbitmq secret engine.

This creates an secret engine of type `rabbitmq` at path `/rabbitmq`

``` bash
curl --request POST http://127.0.0.1:8200/v1/sys/mounts/rabbitmq \
    --header "X-Vault-Token: root" \
    --data-raw '{
  "type": "rabbitmq"
}'
```

### Connect vault and rabbit

Now we have enabled the secret engine, its not very useful unless we tell vault how to talk to rabbitMQ.
As we are using docker compose to host the services we can make use of the dns that comes with it that allows us to just call the service name to talk between containers.

As for the username and password rabbitmq comes with an admin user out the box, so we can just use that to authenticate.

``` bash
curl --request POST http://127.0.0.1:8200/v1/rabbitmq/config/connection \
--header 'X-Vault-Token: root' \
--data-raw '{
    "connection_uri": "http://rabbit:15672",
    "username": "guest",
    "password": "guest"
}'
```

### Create rabbit role

Next we need to tell vault what the rabbitMQ role should look like, this will be the permission in side of rabbitMQ that the new user will get.

These permissions will let the app configure and services called chat, and auto named services. [https://www.rabbitmq.com/access-control.html#authorisation](https://www.rabbitmq.com/access-control.html#authorisation)

This create a role at `/chat` with the access needed to send and resive messages. The name of the role is the same as what we gave access to from the vault policy earlyer.

``` bash
curl --request POST http://127.0.0.1:8200/v1/rabbitmq/roles/chat \
    --header "X-Vault-Token: root" \
    --data-raw '{
  "vhosts": "{\"/\": {\"configure\":\"^(chat|amq.gen.*)$\", \"write\":\"^(chat|amq.gen.*)$\", \"read\": \"^(chat|amq.gen.*)$\"}}",
  "vhost_topics": "{\"/\": {\"chat\": {\"write\":\".*\", \"read\": \".*\"}}}"
}
'
```

## Add code to get rabbitMQ details

Time to put it all together and add the code to let our application get the rabbitMQ login.

Add the following code after the `//code to log in to rabbit` comment

``` go
  client := &http.Client{}
  req, err := http.NewRequest("GET",fmt.Sprintf("%s/v1/rabbitmq/creds/chat", vaulturl),nil) //bulds the url to the secrets endpoint
  failOnError(err, "failed make req")
  req.Header.Set("X-Vault-Token",up.Auth.ClientToken) //sets the token to the vault returned from the last block
  resp, err = client.Do(req) //sends the HTTP request
  failOnError(err, "failed to send request")
  defer resp.Body.Close()
  body, err = io.ReadAll(resp.Body)
  failOnError(err, "failed to read body")
  fmt.Print(string(body))
  var rab rabbit
  err = json.Unmarshal(body,&rab) //converts the body in to a go struct containing the rabbitMQ username and password
  failOnError(err, "failed to Unmarshal body")
  fmt.Println(rab)
  runRabbit(rab.Data.Username,rab.Data.Password) //runs the rabbit code passing though the username and password retreaved from vault
```
