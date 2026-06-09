
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
