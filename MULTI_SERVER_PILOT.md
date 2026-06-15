Pilot plan — simple steps (for non-technical review)

Goal: Validate monitoring end-to-end on 2 apps across 2 servers (one Ionos, one Plesk) for 72 hours.

What we will test:
- Collect metrics (uptime, CPU, response time)
- Collect logs (basic error events)
- Run simple synthetic HTTP check (homepage + login flow)
- SSL expiry monitoring
- Alerting to Slack/email on site-down or SSL expiring

Very simple pilot steps (what I will prepare now):
1. Inventory template (`server_inventory.csv`) — add your servers there.
2. A short runbook explaining what credentials are needed and how to provide them safely.
3. Example install scripts / Ansible playbook that *you* can run (no secrets inside). 
4. Simple alert rules (site down, slow response > X ms, SSL expiring in 14 days).

What requires credentials or access (I cannot accept secrets in chat):
- SSH keys or sudo access to install agents
- Plesk admin/API credentials to query panel
- Ionos control panel API keys for automation

Safe ways to share credentials:
- Best: Put secrets into your Vault (HashiCorp/AWS/Azure) and give me the Vault path OR
- Provide a temporary SSH public key to add to the server for a short window OR
- If you prefer, I prepare scripts and you run them locally (I won’t need credentials)

Choose one option now:
- A) I prepare templates, scripts, and the runbook you can run (no creds shared).
- B) You will provide Vault path or temporary access (tell me which), and I will run the minimal onboarding steps.

Reply with `A` or `B` and, if B, how you will share access (Vault path or temporary SSH key).