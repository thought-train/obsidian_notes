
1) since we're making with go, do `go init` such that go can make go.mod file and add all dependencies with project name and everything.
2) our dependencies would be azure blob storage(`azblob`), azure table storage(`aztables`) and `azqueue` to retrieve from storages of azure and manage storage queues, and to send and receives requests. 
3) now, we make the go server part, i.e `main.go` which is a simple HTTP client, with post requests and other general endpoint handling templates, typing with azure foundry.
4) this main.go client will then be called by each service for diff tasks, i.e. flux.go for image generation and gpt.go for initial prompt generation and caption generation. All of thes ego files go into a package called foundry which you create, whcih wil then be used by main.go for general HTTP request and response stuff.
	1) flux.go returns 2 kinds of image aspect ratios: 4:5 for posts and 9:16 for stories. this implements the same functionality as the curl request and flux_generate.py
	2)  gpt.go does 2 tasks: initial prompt gen and final caption gen. this receives the response as json and converts it into typed structs in go
	3) sora.go takes in an existign image and makes an mp4 out of it. TODO: update endpoint url
	4) blob.go handles uploading of images and videos to blob storage, and gen and storing sas links.
	5)  table.go handles adding entry to Azure Table Storage, so that it can retrieved through json to dashboard:
		1)  PartitionKey = feed_type (story or post), so all posts will be stored in the same server and all stories in another.
		2) RowKey = post/story uuid
		3) other fields check in the pipeline doc.
5)  Now, we make Python function workers which acts as Triggers for the Azure functions, like background jobs in Azure functions. For this we need requirements.txt and host.json
6) ```                                                              
    generation_trigger fires                                                                
      → generate.py: GPT picks content type + writes FLUX prompt                            
      → FLUX.2-pro draws the image (PNG bytes)                                              
      → image saved to Blob Storage: raw-images/{session}/{uuid}.png                        
      → queue message dropped: "new image ready, here's the blob name + prompt"             
  Immediately after (queue trigger fires per message)                                       
    caption_trigger fires                                                                   
      → caption.py: GPT reads the FLUX prompt → writes caption + 25 hashtags                
      → Table Storage: new row PartitionKey="feed"|"story", status="pending"                                                                                 
    posting_trigger fires                                                                   
      → post.py: queries Table Storage for status="approved"                                
      → generates a 15-min SAS URL for the image blob                                       
      → calls Instagram API: create container → publish                                     
      → updates Table Storage: status="posted", saves instagram_post_id            
	```

## azure infra creation
#### storage account
- we need to create a storage account(*luminatestorage5264*) for our resource group first so that we have a place to store images, reels, stories, captions, prompts, status of each photo, reasons and everything else.
- first, i went to my resource group, `rg-team-lum-corp-5264-resource`, clicked create, then storage account(which has blob, table and queue), then name, resource group and *locally-redundant storage* and created.
- now we create each container in this :
	- Blob :
		- `post-images` container for all 4:3 images
		- `story-images` container for all 16:9 images
		- `reels` container for all video posts and reels
		- `reference-images` container to maintain consistency across all images(TODO this can be tied into FLUX.2 pro model as that image model can accept multiple image inputs to be used as input, should look into whether sora-2 can do that)
	- Table:
		- `posts` container to store the table metadata with PartitionKey as type of content and RowKey as status
	- Queues:
		- `caption-jobs` container so that it can generate captions after ssing an image has been created and notification has been sent to this queue, from which the caption worker can read.
#### container registry
- we need this as azure needs a place to pull opur go dashboard image from and deploy it in container apps.
- while creating, i kept the plan as basic as opposed to standard due to cost($5 vs $20/mo) and that i don't need other customers  as users of this registry and one admin user would suffice for it to only be deployed to the container apps service of azure.
- further, go to settings->access keys and enable admin user and copy the password somewhere safe.
- now, i ned to push my own docker container to the registry with docker build -t and docker push. I logged in with docker login godashboardcr.azurecr.io, built with tag godashboardcr.azurecr.io/godashboard:latest, and pushed it. 
### log analytics workspace
- another prerequisite to going the container apps route is yopu need to have log analytics workspace to create it, where it can dump the container's logs.
- for this, i went to the resource gropup, created a log analytics workspace, and can choose this whiole creating the container app .(previously we needed to note down the *workspace id* and *primary key* to be used in the container apps registration process, at my time in providence)
#### container app
- now, i went to create container app, specified name `godashboard`, also made a godashboard env as it is the cluster in which this docker container will run, and finally selected ingress as external, http and port as 8080.
- I also need to add my env variables in this like:
	- AZURE_API_KEY
	- FLUX_ENDPOINT
	- GPT_ENDPOINT
	- SORA_ENDPOINT
	- AZURE_STORAGE_CONNECTION_STRING(which you get form the storage account we created previously)
	- 