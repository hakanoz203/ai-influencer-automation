# Autonomous AI Influencer Ecosystem: Project "Selinger"

## 📌 Executive Summary
Project "Selinger" is a fully autonomous, almost zero-touch content generation and social media marketing ecosystem. It is designed to operate as a self-sustaining digital entity: a sarcastic, brutally honest relationship analyst. 

The system continuously monitors and analyzes Twitter Trending Posts, Competitor Youtube channels, generates highly engaging multimedia content(including instagram reels videos, X Posts), handles rate-limit exceptions without crashing, and chats followers about their relations ship issues on Instagram DM and drives organic traffic directly to an automated sales funnel (Instagram DM Automation) for a digital product (the GÖRÜLDÜ ATILMADI, TERK EDİLDİN e-book).

## 🏗️ System Architecture & Core Pipelines

The ecosystem is not a linear script; it is a **fault-tolerant state machine** orchestrated via n8n, consisting of several interdependent pipelines.

### 1. 🧠 Master Orchestrator Workflow
The operational brain of the system. Triggered by a strict cron schedule, this workflow dictates the daily production cycle.
*   **Error Handling:** It monitors the production queues. If external data extraction fails (due to API limits or lack of competitor posts), the Error handler detects the empty state and routes the system to the **Evergreen Fallback Pipeline**, ensuring 100% uptime and zero blank days on social media.

### 2. 🕵️ YouTube Competitor Analysis Pipeline
The primary data ingestion engine.
*   **Operation:** Scrapes targeted competitor YouTube channels last 24h videos's transcripts using Supadata.
*   **Processing:** Extracts video transcripts and feeds them to the GPT-4o Persona Engine.
*   **Output:** Transforms competitor insights into Selin's signature sarcastic tone, producing raw video hooks, video scripts, and SEO-optimized descriptions.

### 3. 🐦 Twitter/X Scraper & Skandal Pipeline
The real-time trend hijacking module.
*   **Operation:** Scrapes viral relationship/dating scandal tweets from Twitter/X.
*   **Processing:** Identifies high-engagement "scandals" or toxic relationship stories.
*   **Output:** Generates aggressive, polarizing video hook, video script for video generation and tweet posts designed to maximize algorithmic reach and provoke audience interaction.

### 4. 🌲 Evergreen Lore Pipeline (The Fallback)
The system's built-in Chaos Engineering defense mechanism.
*   **Purpose:** Acts as a "Plan B" when external APIs fail (e.g., HTTP 429 Too Many Requests) or competitors are inactive.
*   **Content:** Triggers a database of pre-defined timeless relationship advice and toxic scenarios, independently generating content from scratch without needing external input.

### 5. 🎬 Video Generation & Distribution Pipeline
The multimedia assembly line.
*   **Operation:** Takes the finalized scripts and generates voiceovers and videos.
*   **Linguistic Override:** Incorporates hard-coded prompt engineering to prevent TTS (Text-to-Speech) hallucination.
*   **Distribution:** Automatically packages the multimedia assets and sends them to a centralized Notion database for archiving, and to Telegram/Social channels for publishing.

### 6. 💬 Instagram DM Automation (includes The Sales Funnel)
Followers can chat about their relationship issues with Selinger via Instagram DM's
It is also a monetization endpoint where audience engagement converts to sales.
*   **Keyword Automation:** Listens for specific trigger words ("KİTAP") in comments or DMs, instantly replying them with a personalized hook text and a direct link to the *GÖRÜLDÜ ATILMADI, TERK EDİLDİN* e-book via DM's.
*   **Conversational AI:** Handles chat automation about relationship issues in DM's, simulating the "Selinger" persona in direct messages to warm up leads before pushing the product link.

## 🛡️ Security & Fault Tolerance

This system was built with the mindset that **"everything will fail."**
*   **Strict JSON Sanitization:** The code nodes feature custom Regex algorithms (`aiText.replace(/
```json/gi, '').trim()`) to strip markdown artifacts and prevent pipeline crashes when the LLM hallucinates (e.g., generating Turkish characters like `açıklama` instead of `aciklama` in JSON keys).
*   **Zero Trust Architecture:** The application does not expose any inbound ports. The n8n instance lives inside an isolated Docker container on a Hostinger VPS and communicates with the outside world exclusively through a **Cloudflare Tunnel**, providing immediate SSL encryption and DDoS protection.
*   **Secrets Management:** API keys (OpenAI, Supadata, etc.) are strictly kept out of environmental variables and are injected directly into n8n's Encrypted Credentials Vault.

## 🚀 Deployment

The system is deployed via Infrastructure as Code (IaC) using Docker Compose.

1. Clone the repository (Note: `.env` is intentionally ignored via `.gitignore`).
2. Create your `.env` file using the provided template:
   ```bash
   cp .env.example .env

