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

# Melih Publish
## Tech Stack
* n8n Automation: Core workflow automation and orchestration on trend analysis, generating prompts, image and video generation.
* Runpod + ComfyUI: Image and video generation using WAN models backend with strict consistent character.
* Qwen image edit Ostris AI Toolkit: Models for maintaining character consistency.
* Automated Social media publishing (not certain).

# Can Publish
## Prerequisites
Before running this workflow, ensure you have the following set up:
* An active n8n instance (local or cloud).
* ComfyUI installed with necessary custom nodes,WAN,QWEN and flux models.
*Before starting this automation, you must prepare your character's LoRA model by following these steps:
1. Generate your character dataset using the Character Creator workflow in ComfyUI.
2. Train a LoRA model with this dataset using the Ostris AI Toolkit via RunPod Docker.
3. Move the resulting trained model into your `ComfyUI/models/loras` directory.

## Setup & Installation

1. **Clone the Repository**
   ```bash
   git clone https://github.com/hakanoz203/ai-influencer-automation
   cd ai-influencer-automation

   ## Version 1.0
   # AI Influencer Content Factory

"Harsh Relationship Analyst" nişi için uçtan uca otonom içerik üretim otomasyonu. n8n + Reddit/Apify + Notion + Gemini/GPT-4o.

