---
title: "Managing API Secrets in Github Pages"
date: '2022-06-29'
categories:
  - miscellaneous
tags: 
- html
- github
classes: 
- wide
- text-justify
---

# Context
I recently realized that I had been leaving an API key exposed (oh no!) in the html code for some of my github pages websites. Trying to find how to manage this, however, led me to see that the solution was not quite as straightforaward as I hoped, and I couldn't find much help on Stackoverflow. The process involves creating a custom build and deploy workflow (yikes!). I hope this guide saves someone else some confusion.

I'll also add, that changes the visibility of the repo is not an option -- github only allows you to restrict access to your github pages if you have an enterprise account.

# First Step (the obvious part):
Github allows you to set secret environmental variables. You can find this in your repository's settings under `Security>Secrets and Variables>Actions`. Add you API key here as a secret variable

<!-- <iframe src="/assets/images/same_r_random_dir_plotly.html" height="600px" width="150%" style="border:none;"></iframe> -->
![](/assets/images/github_pages_post/set_secret_variable.png)


# Second Step (where things get weird)
So, setting the secret variable is easy, but actually retreiving it is where things get complicated. To actually be able to retreive this secret variable, we need to create a custom build and deploy workflow and set the variable there. To do so, navigate to `settings>pages`, change the source to `"github actions"`, then click on `"create your own"`.


![](/assets/images/github_pages_post/created_workflow.png)

This will create a file in a folder name `.github/workflows`. You can name this file something like `build_and_deploy_with_secrets.yaml`.
Paste this code in there and savit it:

```yaml
# Build and deploy workflow

name: Build and Deploy
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:
  # Allows you to run this workflow manually from the Actions tab
permissions:
  contents: write


# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains one job with two main steps: build and deploy
  build-and-deploy:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Checkout
        uses: actions/checkout@v3
        env: # Set Secret Key
          secret_adobe_api_key: $\{{ secrets.ADOBE_API_KEY }\}

      # I believe the site is pre-built, otherwise an install/build step goes here

      - name: Deploy 🚀
        uses: JamesIves/github-pages-deploy-action@v4
        # https://github.com/JamesIves/github-pages-deploy-action
        with:
          folder: . # build # The folder the action should deploy.
                    # A . signifies the root of the directory
```

The important part for this workflow are these two lines:

```yaml
env: # Set Secret Key
    secret_adobe_api_key: $\{{ secrets.ADOBE_API_KEY }\}
```

Note that `ADOBE_API_KEY` is what we called the secret variable in step one, and `secret_adobe_api_key` is what we're going to retreive in the next step. After pushing this file to main, you can check to make sure that your page still builds and deploys in the actions tab of the github repo. 

# Third Step (Finish it up!)
Everything is now setup for us to change out our secret value for an env variable. Let's change the hardcoded value in our `index.html` to be a variable called `api_key`. Note, `secret_adobe_api_key` is the variable we set in step 2, not the secret variable we set in step 1. This snippet is some javascript code in my html file that I use to open a pdf within a webpage.

```html
<script type="text/javascript">
	api_key = process.env.secret_adobe_api_key
	document.addEventListener("adobe_dc_view_sdk.ready", function(){ 
		var adobeDCView = new AdobeDC.View({clientId: api_key, divId: "adobe-dc-view"});
		adobeDCView.previewFile({
			content:{location: {url: "https://raw.githubusercontent.com/jthaller/resume_url_hosting/32236b74a33e69f6e12ffd6354e38c95ddcc5ed4/Jeremy_Thaller_Resume.pdf"}},
			metaData:{fileName: "Jeremy_Thaller_Resume.pdf"}
		}, {});
	});
</script>
```
