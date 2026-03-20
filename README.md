# Project 3 — Automated Daily Infrastructure Report with AI Summary

## Overview

This project builds a fully automated daily infrastructure reporting system.
Every morning at 8:00 AM, the workflow collects live server health data from
an AWS EC2 instance, passes it through a Groq AI agent to generate a professional
written summary, then delivers the report as a formatted HTML email via Gmail
and posts it to a Slack channel — all without any manual intervention.

**New skills learned in this project:**

- Schedule Trigger (cron-based automation)
- HTTP Request node (calling custom endpoints)
- AI summarisation with Groq (LLaMA 3.3 70B)
- HTML email formatting
- Gmail OAuth2 credential setup
- Slack message delivery
- Node.js health server with pm2 process management

---

## Architecture

```
┌──────────────────┐
│ Schedule Trigger │  Fires every day at 8:00 AM
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  HTTP Request    │  Calls health endpoint on AWS server
│  (GET /health)   │  Returns CPU, RAM, Disk, Uptime as JSON
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Code Node       │  Formats raw server data into
│  (Prompt Build)  │  a structured AI prompt
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  HTTP Request    │  Sends prompt to Groq API
│  (Groq AI)       │  Model: llama-3.3-70b-versatile
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Code Node       │  Extracts summary text from
│  (Extract)       │  Groq response, strips markdown
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌───────┐
│ Gmail │  │ Slack │
│ HTML  │  │ Post  │
│ Email │  │       │
└───────┘  └───────┘
```

---

## Prerequisites

Before building this workflow, ensure you have the following ready:

- A running n8n instance (self-hosted or cloud)
- An AWS EC2 instance running Ubuntu with Node.js installed
- A Groq account with an API key from `console.groq.com`
- A Gmail account with Google Cloud OAuth2 credentials configured
- A Slack workspace with a dedicated channel (e.g. #devops)
- Port 3000 open in your AWS Security Group inbound rules

---

## Part 1 — Health Server Setup on AWS

### What is the Health Server?

The health server is a small Node.js program that runs permanently on your AWS EC2 instance. It listens on port 3000 and responds to HTTP GET requests with live server metrics in JSON format. When n8n calls it every morning, it returns the following data:

- CPU usage percentage
- RAM used, free, and total (in MB)
- Disk usage
- Server uptime
- Hostname

Think of it like hiring a receptionist inside your server. Every time someone knocks on the door and asks "how are you doing?", the receptionist collects the server metrics and hands them back as structured JSON data.

---

### Step 1 — SSH into Your Server

Open MobaXterm and connect to your AWS EC2 instance using your server's public IP address.

---

### Step 2 — Check Node.js is Installed

```
node --version
```

You should see a version number like `v18.19.1`. If you see `command not found`, install Node.js first before proceeding.

---

### Step 3 — Create the Health Server File

Run this command to create the file without opening any editor:

```
cat > health-server.js << 'EOF'
const http = require('http');
const os = require('os');
const { execSync } = require('child_process');

http.createServer((req, res) => {
  if (req.url === '/health') {
    const totalMem = os.totalmem();
    const freeMem = os.freemem();
    const usedMem = totalMem - freeMem;

    let disk = 'N/A';
    let cpu = 'N/A';
    let uptime = 'N/A';

    try {
      const dfOut = execSync("df -h /").toString().split('\n')[1].trim().split(/\s+/);
      disk = dfOut[2] + '/' + dfOut[1] + ' (' + dfOut[4] + ' used)';
    } catch(e) {}

    try {
      cpu = execSync("grep 'cpu ' /proc/stat | awk '{usage=($2+$4)*100/($2+$4+$5)} END {print usage\"%\"}'").toString().trim();
    } catch(e) {}

    try {
      uptime = execSync("uptime -p").toString().trim();
    } catch(e) {}

    const data = {
      timestamp: new Date().toISOString(),
      cpu_usage_percent: cpu,
      memory: {
        total_mb: Math.round(totalMem / 1024 / 1024),
        used_mb: Math.round(usedMem / 1024 / 1024),
        free_mb: Math.round(freeMem / 1024 / 1024),
        usage_percent: Math.round((usedMem / totalMem) * 100)
      },
      disk: disk,
      uptime: uptime,
      hostname: os.hostname()
    };

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(data, null, 2));
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}).listen(3000, () => {
  console.log('Health server running on port 3000');
});
EOF
```

---

### Step 4 — Start the Server and Test It

Start the server in the background:

```
node health-server.js &
```

Test it locally:

```
curl http://localhost:3000/health
```

You should see a JSON response like this:

```
{
  "timestamp": "2026-03-17T08:00:00.000Z",
  "cpu_usage_percent": "0.42%",
  "memory": {
    "total_mb": 911,
    "used_mb": 734,
    "free_mb": 177,
    "usage_percent": 81
  },
  "disk": "5.6G/6.8G (84% used)",
  "uptime": "up 4 hours, 7 minutes",
  "hostname": "ip-172-31-41-60"
}
```

---

### Step 5 — Make the Server Permanent with pm2

The `&` symbol keeps the server running temporarily but it stops when 
you close the terminal. To keep it running permanently install pm2:

```
sudo npm install -g pm2
pm2 start health-server.js
pm2 startup
pm2 save
```

After running `pm2 startup`, copy and run the command it outputs — it will 
look something like `sudo env PATH=...`. Then run `pm2 save` to preserve 
the process across reboots.

Verify it is running:

```
pm2 list
```

You should see `health-server` listed with status **online**.

---

### Step 6 — Open Port 3000 on AWS

Your server's firewall blocks port 3000 by default. To allow n8n to reach the health endpoint from outside:

1. Go to **AWS Console** → **EC2** → **Security Groups**
2. Click your instance's security group
3. Click **Inbound Rules** → **Edit inbound rules**
4. Click **Add Rule** and fill in:

| Field | Value |
|---|---|
| Type | Custom TCP |
| Port range | 3000 |
| Source | 0.0.0.0/0 |

5. Click **Save rules**

---

### Step 7 — Verify Public Access

Open a browser and go to:

```
http://YOUR_SERVER_PUBLIC_IP:3000/health
```

You should see the same JSON response in the browser. This confirms n8n can reach the endpoint from anywhere.

---

## Part 2 — n8n Workflow Setup

### Node 1 — Schedule Trigger

| Field | Value |
|---|---|
| Trigger Interval | Days |
| Days Between Triggers | 1 |
| Trigger at Hour | 8am |
| Trigger at Minute | 0 |

---

### Node 2 — HTTP Request (Health Endpoint)

| Field | Value |
|---|---|
| Method | GET |
| URL | `http://YOUR_SERVER_IP:3000/health` |
| Authentication | None |

---

### Node 3 — Code Node (Format Prompt)

This node takes the raw server health JSON and formats it into a structured prompt for the AI:

```javascript
const data = $input.first().json;

return [{
  json: {
    model: "llama-3.3-70b-versatile",
    max_tokens: 1000,
    temperature: 0.4,
    messages: [
      {
        role: "user",
        content: `You are an infrastructure monitoring assistant. Analyze the following
server health data and write a professional daily infrastructure report in 3-4 paragraphs.
Highlight any concerning values — memory above 80%, disk above 85%, or CPU above 80%.

Server Health Data as of ${data.timestamp}:
- Hostname: ${data.hostname}
- CPU Usage: ${data.cpu_usage_percent}
- Memory: ${data.memory.used_mb}MB used out of ${data.memory.total_mb}MB (${data.memory.usage_percent}% used)
- Disk: ${data.disk}
- Uptime: ${data.uptime}`
      }
    ]
  }
}];
```

---

### Node 4 — HTTP Request (Groq AI)

| Field | Value |
|---|---|
| Method | POST |
| URL | `https://api.groq.com/openai/v1/chat/completions` |
| Authentication | Generic Credential Type → Header Auth |
| Header Name | `Authorization` |
| Header Value | `Bearer YOUR_GROQ_API_KEY` |
| Specify Body | Using JSON |
| Body | `={{ JSON.stringify($json) }}` |

---

### Node 5 — Code Node (Extract Summary)

This node extracts the AI-generated text and strips markdown formatting symbols:

```javascript
const response = $input.first().json;

const raw = response.choices[0].message.content;

const summary = raw
  .replace(/\*\*(.*?)\*\*/g, '$1')
  .replace(/#{1,6}\s/g, '')
  .replace(/\*(.*?)\*/g, '$1')
  .trim();

return [{
  json: {
    summary: summary,
    timestamp: new Date().toLocaleString('en-GB', { timeZone: 'Africa/Lagos' }),
    hostname: "ip-172-31-41-60"
  }
}];
```

---

### Node 6 — Gmail (HTML Email)

| Field | Value |
|---|---|
| Credential | Gmail OAuth2 account |
| Resource | Message |
| Operation | Send |
| To | your-email@gmail.com |
| Subject | `Daily Infrastructure Report — {{ $json.timestamp }}` |
| Email Type | HTML |

HTML body to paste into the Message field:

```html
<div style="font-family: Arial, sans-serif; max-width: 650px; margin: auto; border: 1px solid #ddd; border-radius: 8px; overflow: hidden;">
  <div style="background-color: #1A6496; padding: 24px;">
    <h1 style="color: white; margin: 0; font-size: 22px;">Daily Infrastructure Report</h1>
    <p style="color: #cce5f6; margin: 6px 0 0;">{{ $json.timestamp }} — {{ $json.hostname }}</p>
  </div>
  <div style="padding: 24px; background: #ffffff;">
    <p style="color: #333; line-height: 1.8; white-space: pre-line;">{{ $json.summary }}</p>
  </div>
  <div style="background: #f4f4f4; padding: 16px; text-align: center;">
    <p style="color: #999; font-size: 12px; margin: 0;">Automated report generated by your n8n infrastructure monitor</p>
  </div>
</div>
```

---

### Node 7 — Slack

| Field | Value |
|---|---|
| Credential | Slack account |
| Resource | Message |
| Operation | Send |
| Channel | #devops |
| Message Text | `{{ $json.summary }}` |

---

## Part 3 — Gmail OAuth2 Setup

### What is OAuth2 and Why Does Gmail Require It?

Google shut down direct password access for third-party apps in 2022. 
Any app that wants to send emails through Gmail must now go through 
Google's official OAuth2 process. This means the app never sees your
password — Google handles authentication and issues a secure token instead.

Think of it like a hotel key card system. Your actual password is the master
key that you never hand to anyone. OAuth2 is the front desk issuing a
temporary key card that only opens specific doors and can be cancelled at any time.

---

### Step 1 — Create a Google Cloud Project

1. Go to `console.cloud.google.com`
2. Click the project dropdown at the top
3. Click **New Project**
4. Name it `n8n` and click **Create**

---

### Step 2 — Enable the Gmail API

1. Go to **APIs & Services** → **Enable APIs and Services**
2. Search for **Gmail API**
3. Click it and click **Enable**

---

### Step 3 — Configure the OAuth Consent Screen

1. Go to **Google Auth Platform** → click **Get started**
2. Fill in **App name** as `n8n`
3. Fill in **User support email** with your Gmail address
4. Fill in **Developer contact email** with your Gmail address
5. Click through all steps until completion
6. Go to **Audience** → click **Add users**
7. Add your Gmail address as a test user
8. Click **Save**

---

### Step 4 — Create OAuth Credentials

1. Go to **Clients** → click **Create OAuth client**
2. Select **Web application** as the application type
3. Name it `n8n`
4. Under **Authorised redirect URIs** click **Add URI** and paste:

```
https://YOUR_N8N_DOMAIN/rest/oauth2-credential/callback
```

5. Click **Create**
6. Copy the **Client ID** and **Client Secret** from the popup

---

### Step 5 — Connect Gmail in n8n

1. Open the Gmail node in your n8n workflow
2. Click **Create new credential**
3. Paste the **Client ID** and **Client Secret**
4. Click **Save**
5. Reopen the credential and click **Sign in with Google**
6. Authorise with your Gmail account

Once you see **Account connected** in green, the Gmail credential is ready.

---

## Key Lessons Learned

**OAuth2** is Google's way of letting third-party apps use your Gmail without
ever seeing your password. It is the industry standard for secure app authorisation.

**pm2** keeps Node.js processes alive permanently on a Linux server. Without it, 
any Node.js script stops running the moment you close your terminal session.

**n8n branching** allows one node to send data to multiple nodes simultaneously. 
To create a branch, hover over the right edge of a node until the connection dot 
appears, then drag it to the target node.

**Groq API** is OpenAI-compatible, meaning you can use it exactly like OpenAI in
any tool that supports OpenAI — just change the base URL and API key.

**Health endpoints** are a standard pattern in infrastructure monitoring. Rather 
than SSHing into a server to check its status, you expose a `/health` endpoint that
returns structured data any monitoring tool can consume.

---

## Screenshots to Include

**1. n8n Workflow Canvas**

A full screenshot of the complete workflow showing all 7 nodes connected 
in sequence — Schedule Trigger, HTTP Request, Code, Groq HTTP Request, 
Code, Gmail, and Slack.

**2. Health Endpoint in Browser**

A screenshot of the browser showing the JSON response from 
`http://YOUR_SERVER_IP:3000/health` with all fields populated
— CPU, memory, disk, uptime, and hostname.

**3. Terminal — pm2 Process List**

A screenshot of the MobaXterm terminal showing the output of `pm2 list`
confirming the health server is running and online.

**4. Gmail Inbox**

A screenshot of the received HTML email in Gmail showing the blue header, 
timestamp, hostname, and the AI-generated infrastructure report body.

**5. Slack Channel**

A screenshot of the #devops Slack channel showing the posted infrastructure
report message with the full AI-generated summary.

---

## Troubleshooting

**Health endpoint not responding**
- Cause: Server stopped or port closed
- Fix: Run `pm2 restart health-server` and check AWS Security Group inbound rules

**Gmail OAuth error 414**
- Cause: Credential not saved before authorising
- Fix: Save the credential first, then click Sign in with Google

**Gmail access blocked**
- Cause: Email not added as test user in Google Cloud
- Fix: Go to Google Auth Platform → Audience → Add test users → add your Gmail

**Groq returning undefined**
- Cause: Wrong node connected to Slack
- Fix: Ensure Slack node receives input from Code in JavaScript1

**Markdown symbols in report**
- Cause: AI returning formatted text with asterisks and hashes
- Fix: Use `.replace()` in the Code node to strip `**` and `###` from the summary

---

## What's Next — Project 4 Preview

The next project will build on this foundation by adding multi-server 
monitoring — collecting health data from multiple servers simultaneously,
comparing their status, and generating a consolidated report that highlights
which servers need attention.
