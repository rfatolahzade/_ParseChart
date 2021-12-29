
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
Here you go.
