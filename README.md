## Can Publish
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
