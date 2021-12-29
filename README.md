![Parse Evolution](images/logo.jpg)

# Parse Evolution
Check docker-compose config:
```bash
git clone https://github.com/rfinland/parse.git
cd parse
docker-compose --env-file .env config 
```
Notice: Application-Id values are selective with user MASTER_KEY; you can set values for these envs as you want.
In our case APP_ID=MyParseApp and MASTER_KEY=adminadmin in parse 
Up the app:
```bash
docker-compose --env-file .env up -d
#OR If already exist:
docker-compose --env-file .env up --force-recreate -d
```
#notice: If you want to create kube via docker-compose yaml file you have to set your envs as static in your yaml file
#Check Parse status
```bash
curl http://localhost:1337/parse/health
#Result: {"status":"ok"}
```
# POST 
Post a record:
You have to set your X-Parse-Application-Id as you set in .env file (in our case:APP_ID=MyParseApp)
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:1337/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"ngaYmcDv1m",
"createdAt":"2021-12-06T06:56:09.068Z"
}
```
# GET
Get the record:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore/ngaYmcDv1m
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:1337/parse/classes/GameScore
```
Also you can see the result in your browser (you just need set X-Parse-Application-Id: MyParseApp )
# Recommended add-ons
##### JSON Formatter: 
Chrome extension for printing JSON and JSONP nicely when you visit it 'directly' in a browser tab.
[JSON Formatter](https://github.com/callumlocke/json-formatter)

##### Modify Header:
Add, modify or remove a header for any request on desired domains.
[Modify Header](https://mybrowseraddon.com/modify-header-value.html)
##### RestMan:
RESTMan is a browser extension to work on http requests.
[RestMan](https://chrome.google.com/webstore/detail/restman/ihgpcfpkpmdcghlnaofdmjkoemnlijdi)

#DELETE Records reamin
#How to delete records that we created.
Lets do these steps with Kubernetes:
```bash
docker-compose --env-file .env down
```
kompose Installation:
# Linux
```bash
curl -L https://github.com/kubernetes/kompose/releases/download/v1.25.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
#OR
brew install kompose
```

# Install k3s
Install your kube, in our case k3s:
```bash
curl -sfL https://get.k3s.io/ | sh -s - --docker 
```
add export kubeconfig to ~/.bashrc:
```bash
cat <<EOF >> ~/.bashrc
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
EOF
```
and then run :
```bash
source .profile
```

Add these usefull aliases to .bashrc:
```bash
cat <<EOF >> ~/.bashrc
alias k='kubectl'
alias kga='watch -x kubectl get all -o wide'
alias kgad='watch -dx kubectl get all -o wide'
alias kcf='kubectl create -f'
alias wk='watch -x kubectl'
alias wkd='watch -dx kubectl'
alias kd='kubectl delete'
alias kcc='k config current-context'
alias kcu='k config use-context'
alias kg='k get'
alias kdp='k describe pod' 
alias kdes='k describe'
alias kdd='k describe deployment'
alias kds='k describe svc'
alias kdr='k describe replicaset'
alias kk='k3s kubectl'
alias vk='k --kubeconfig ./kubeconfig.yaml'

EOF
```
and then run:
```bash
exec bash
```
# Convert time
just run :
```bash
kompose convert
#OR 
kompose convert -o parse.yaml
```
If you ran "kompose convert" you have to move your outputs in subdirectory as below:
```bash
mv mongo-service.yaml mongo-deployment.yaml server-deployment.yaml server-service.yaml kube
cd kube
kubectl apply -f .
```
OR If you ran "kompose convert -o parse.yaml" then just run:
```bash
kubectl apply -f parse.yaml
```
Two way to find out why your pod not ready (if it happens):
```bash
#First one:
k logs pod/server-74cd8bb94d-bhtwt   #your pod name
#Second one:
kdes pod/server-74cd8bb94d-bhtwt   #describe of your not ready pod
```
To access to your Parse:
```bash
k port-forward service/server 1337:1337
```
Now lets expose this port via our yaml file:
You have to add "nodePort: 30001" & type: NodePort under spec section of server service:
```bash
spec:
.
.
      targetPort: 1337
      nodePort: 30001
  selector:
    io.kompose.service: server
  type: NodePort
status:
  loadBalancer: {}

```
Now lets POST a sample record:
```bash
curl -X POST \
-H "X-Parse-Application-Id: MyParseApp" \
-H "Content-Type: application/json" \
-d '{"score":1000,"playerName":"Rain Man","cheatMode":false}' \
http://localhost:30001/parse/classes/GameScore
```
You should get a response similar to this:
```bash
{
"objectId":"kJx7buQPDW",
"createdAt":"2021-12-08T09:17:08.682Z"
}
Get the record:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore/kJx7buQPDW
```
Get all records that you defined:
```bash
curl -X GET \
  -H "X-Parse-Application-Id: MyParseApp" \
  http://localhost:30001/parse/classes/GameScore
```
done.

# Helming!
```bash
helm create helm-parse
```
First, delete everything under templates directory:
```bash
rm -r helm-parse/templates/*
```
and then copy deployment files(mongo-deployment.yaml,server-deployment.yaml) and service files(mongo-service.yaml,server-service.yaml) from kube and place those files under the templates directory:
```bash
cp $PWD/kube/* $PWD/helm-parse/templates/
```
Lets install our app:
```bash
cd helm-parse
helm install parse .
kga 
```
##### Values.yaml
Lets create/empty values.yaml to set our envs:
```bash
cat <<EOF > values.yaml
server:
 - name: PARSE_SERVER_APPLICATION_ID
   value: MyParseApp
 - name: PARSE_SERVER_MASTER_KEY
   value: adminadmin 
 - name: PARSE_SERVER_DATABASE_URI
   value: mongodb://mongo/test
EOF
```
Now we have to call values inner server-deployment.yaml:
```bash
nano /templates/server-deployment.yaml
```
then modify it as below:
```bash
containers:
        - env:
            {{- range .Values.server }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }} 
          image: parseplatform/parse-server
          name: server
          ports:
            - containerPort: 1337
          resources: {}
      restartPolicy: Always
```
save the file and run helm install contains your values file:	  
```bash
helm install parse . -f values.yaml
kga
```
##### Add ingress 
Let's add ingress to project:
```bash
touch /charts/Parse/templates/ingress.yaml
nano /charts/Parse/templates/ingress.yaml
```
Paste :
```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: rfinland.net
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: server
              port:
                number: 80
```
Take a look to server-services.yaml:
I changed:
```bash
spec:
  ports:
    - name: "1337"
      port: 1337
      targetPort: 1337
      nodePort: 30001
  selector:
```
To:
```bash
spec:
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 1337
  selector:
```
The nodePort removed, and port changed to 80
Apply changes:
```bash
cd /Parse/charts/Parse
helm install parse . -f values.yaml
#helm upgrade parse . -f values.yaml
kg ingress
kga
#On CLUSTER-IP of service/server:
curl 10.43.125.229/parse/health
#On our Ingress:
curl http://rfinland.net/parse/health
```

# Postgres as DB
in this case our docker-compose file will be:
```bash
version: "3.7"

services:
  server:
    image: parseplatform/parse-server
    environment:
      - PARSE_SERVER_APPLICATION_ID=MyParseApp
      - PARSE_SERVER_MASTER_KEY=adminadmin
      - PARSE_SERVER_DATABASE_URI=postgres://postgres:postgres@postgres/postgres
    ports:
       - 1337:1337
    depends_on:
      - postgres
  postgres:
    image: postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=postgres
    ports:
      - 5432:5432
```
I do the same steps as Mongo:
```bash
docker-compose up -d
curl http://localhost:1337/parse/health
```
#### Convert time

```bash
mkdir postgres
cd postgres
cp ../docker-compose.yaml .
kompose convert
rm docker-compose.yaml
kubectl apply -f .
curl http://<CLUSTER-IP>:1337/parse/health
```
#### Helming!
```bash
helm create helm-parse-postgres
rm -r helm-parse-postgres/templates/*
cd postgres
cp $PWD/* $PWD/helm-parse-postgres/templates/
cd helm-parse-postgres
helm install parse .
```
#### Values.yaml
```bash
 cat <<EOF > values.yaml
server:
 - name: PARSE_SERVER_APPLICATION_ID
   value: MyParseApp
 - name: PARSE_SERVER_MASTER_KEY
   value: adminadmin 
 - name: PARSE_SERVER_DATABASE_URI
   value: postgres://postgres:postgres@postgres/postgres
EOF
```
then:
```bash
nano /templates/server-deployment.yaml
```
modify it as below:
```bash
containers:
        - env:
            {{- range .Values.server }}
            - name: {{ .name }}
              value: {{ .value }}
            {{- end }} 
          image: parseplatform/parse-server
          name: server
          ports:
            - containerPort: 1337
          resources: {}
      restartPolicy: Always
```
save the file and run helm install contains your values file:	  
```bash
helm install parse . -f values.yaml
#helm upgrade parse . -f values.yaml
```
##### Add ingress 
Let's add ingress to project:
```bash
touch /templates/ingress.yaml
nano /templates/ingress.yaml
```
Paste :
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: rfinland.net
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: server
              port:
                number: 80
```
Take a look to server-services.yaml and Change it as below:
```bash
spec:
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 1337
  selector:
```
Apply changes:
```bash
helm install parse . -f values.yaml
#helm upgrade parse . -f values.yaml
kg ingress
kga
#On CLUSTER-IP of service/server:
curl 10.43.125.229/parse/health
#On our Ingress:
curl http://rfinland.net/parse/health
```


# GitHub Pages

Create a new branch for github pages:
```bash
#Create a new branch:
git checkout --orphan gh-pages
#We have to make it empty:
git rm -rf .
#Commit & Push the new branch:
git commit --allow-empty -m "root commit"
git push origin gh-pages
#Chech the current branch:
git branch
#Switch to the branch:
git checkout gh-pages
#Switch to master branch:
git checkout master
#Delete branch: When your current branch is main
git branch -d gh-pages
#Delete branch remotely
git push origin --delete gh-pages
```

Lets create an index.html file on our new branch:
```bash
git checkout gh-pages
touch index.html
echo "Hello!" > index.html
git add .
git commit -m "A index html"
git push
```

And then visit:
```bash
https://rfinland.github.io/Parse/index.html
#https://YOURGITHUBUSER.github.io/YOURPROJECT/index.html
#OR just visit:
https://rfinland.github.io/Parse/
```
You should able to see what's happen in github actions on your repo
Also you can set theme for your page and add a README.md
For open source projects, GitHub Pages is a great choice to host Helm repositories. 
We’re using the gh-pages branch to store and serve the packaged charts in this part of article. 
After each release we undergo a manual process of packaging and pushing the new chart version to the gh-pages branch.
Lets add an action.
I added default action named release.yml :
```bash
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.1.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

Run these commands on your machine:
```bash
helm repo add <alias> https://<orgname>.github.io/helm-charts
helm repo add Parse https://rfinland.github.io/Parse
#The index.yaml file:
https://rfinland.github.io/Parse/index.yaml
helm repo list
helm search repo Parse
helm install mongo-parse parse/helm-parse
helm delete mongo-parse
```
