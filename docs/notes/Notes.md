Scope: these notes are for my winterarc project, a personal learning setup. It is not production. Where something is not implemented, the note says so and says what I would do in production.
1. Identity Center vs access keys
IAM Identity Center is the central place to manage who can get into AWS. Users and groups live in one directory. Each group is assigned permission sets for specific accounts, so access is granular. Everyone gets one access portal link, and MFA can be enforced at sign-in.
Access keys are long-lived credentials (access key ID + secret). They never expire until rotated or deleted. If a laptop is lost or a key is committed to Git, whoever holds it can act as that user.
What aws sso login does: opens the browser, I authenticate (with MFA), and the CLI gets a short-lived token. The CLI then exchanges it for temporary role credentials that expire.
* Session durations: there are two separate lifetimes. The role credentials in ~/.aws/cli/cache/ last 8 hours (the permission set session duration, chosen because this is for my own development work). The SSO token cache in ~/.aws/sso/cache/ (I use an sso-session) holds a short-lived access token plus a refresh token. The CLI uses the refresh token to renew the access token without me logging in again, so the cache stays usable for up to 90 days (the access portal session length) until I log out or the session expires.
* Where it's stored: a cached JSON file on my machine. The SSO token lives in ~/.aws/sso/cache/ and the role credentials in ~/.aws/cli/cache/ (the default locations). This cache is what the CLI and SDKs use. It is not what logs me into the web console, which uses the access portal in the browser.
* Why this is better: nothing permanent sits on disk, and credentials expire on their own. A stolen role-credentials file in cli/cache is only useful for up to 8 hours. A stolen sso/cache file is the bigger risk: the refresh token in it can keep requesting fresh role credentials for up to 90 days, so it is the file to protect (aws sso logout clears it).
2. MFA
What it stops: someone logging in with only a password (stolen, leaked, reused, or guessed). A correct password alone is no longer enough.
Three things it does not stop:
1. A stolen session token or access key that was already issued. MFA happens at login, so a token that already exists skips it. The defence is short sessions, not MFA. Mine are 8 hours for role credentials but up to 90 days for the SSO token (see section 1), so the SSO token is the long window. Access keys are worse because they never expire.
2. Real-time phishing proxies (attacker relays my TOTP code as I type it). Only phishing-resistant MFA (passkeys/FIDO2) stops this.
3. Damage from over-broad permissions. Once a legitimate user is in, MFA does nothing about what they can do (see the Admin gap below).
Also not covered: DDoS or attacks against the services I run. Those need other controls (Shield, WAF, etc.).
3. Budgets
Current setup: budgets in the AWS Billing console at $25 / $50 / $75 / $100, with email notification. A budget alert only notifies. It does not stop anything. Services keep running and keep costing money.
Stop mechanism: none right now. Spend is watched, not capped.
Two mechanisms that would actually stop spend (what I'd add):
1. Budget Actions: when a threshold is hit, automatically attach a deny policy (IAM or SCP) or stop specific EC2/RDS instances.
2. SNS alert triggering a Lambda that shuts down or deletes the offending resources.
Billing lag: AWS Budgets data updates up to three times a day, typically 8 to 12 hours apart, and there is a further delay between a charge being incurred and the notification. Spend can pass a threshold before I'm alerted. So an alert is an early warning, not a hard limit.
Why $25 / $50 / $75 / $100: $25 steps up to a $100 ceiling give me four warnings before the ceiling. The amounts are not based on a measured monthly spend (winterarc is a learning project). In production I'd set them from measured spend and decide what action each level triggers.
4. Known gaps
* Day-to-day Admin access is a known gap. I work as a platform engineer under the user John, who has full administrative access. The reason it's this broad: I manage the whole AWS console across several projects and need admin to fix permission problems. That explains why it's convenient, but it is still the gap. A compromised or mistaken admin session can do anything in the account, and MFA doesn't limit what an admin can do once logged in. For winterarc this is an accepted risk. In production I'd use scoped permission sets per project for daily work, and a separate admin permission set with a short session, used only when needed and logged.
* Admin IAM user is break-glass only: allows all administrative work, for high-severity incident response. MFA is enforced on the device. No alert is built: using that user triggers no notification today. The plan is a CloudTrail-based alert with a Slack notification for that specific user, but it does not exist yet. Where its credentials and MFA device are stored is not documented for winterarc. In production they'd be sealed in a vault, with the MFA device held separately.
* Separation of duties: I am the only person on this account. John (daily admin) and the break-glass user are both mine, so there is no second person approving emergency use.
5. Approval problem
What I saw: PRs to main couldn't merge. The repo ruleset required an approval, and as the only person on the repo I can't approve my own PR.
What I changed: set required approvals to 0. Reason: solo repo, no collaborator to approve.
What the ruleset allows now:
* Pull request required before merging to main: yes
* Force push to main: blocked
* Bypass: nobody can bypass, including me as admin
If I add collaborators, I'd set approvals back to 1 or more.