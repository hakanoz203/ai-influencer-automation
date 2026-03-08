# AI Influencer Automation Pipeline
# Hakan publish
A comprehensive n8n-based automation workflow for generating, managing, and publishing content for a consistent AI influencer character. This pipeline handles
the entire lifecycle of a real influencer: Trend Analysis of chosen niche from Google Trend Alerts RSS Feed, Reddit API, Youtube Data API and maybe in the future 
Perplexity API; Generating Prompt for content regarding to trends with ai agents ; Content Creation of images and videos from Runpod + ComfyUI workflows 
connected with API to n8n workflow ; Automated Social Media posting(not certain) ; Recording to database; While maintaining strict character consistency with
LoRa Training processed on Docker Ostris AI Toolkit in RunPod.

# Alperen Publish

Key Features:

* Automated Trend Analysis: Gathers current trends of chosen niche (Google Alerts RSS Feed, Reddit API, Youtube Data API...) and generate data from AI agents to inform content direction.

* Dynamic Prompt Generation: Crafts optimized positive and negative prompts for ComfyUI image and video generation workflows based on analyzed trends.

* Consistent Character Creation: Integrates with ComfyUI and LoRA models(strict consistency provided by Ostris AI Toolkit in Docker) to maintain the influencer's persona's distinct features automated.

* Automated Posting (not certain): Seamlessly schedules and posts generated images and videos directly to Social Media platforms.