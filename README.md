# AWS Cloud Practitioner — Module 7 Storage Role-Play

A 10-minute, written, AI-powered conversation with Olivia (Operations Manager) to assess understanding of EC2 storage, EBS, instance store, EFS, S3, S3 Glacier, backups, and recovery. Includes adaptive questioning, score out of 10, and a downloadable plain-text transcript.

## Architecture

GitHub Pages (`docs/`) → HTTPS POST → Amazon API Gateway HTTP API → AWS Lambda (Node.js 22) → OpenAI API. Lambda retrieves the API key from AWS Secrets Manager. No database. Conversations are sent with each request, and are held only in the current browser page memory (no localStorage). The student downloads their own transcript if desired.

**Important:** This is a formative teaching prototype, not a production-ready authenticated assessment platform. CORS is not authentication; a static access code in `docs/config.js` is visible to students. Before distributing broadly, add genuine identity/authentication (e.g. university SSO/Cognito/LTI), API throttling/WAF, usage budgets, institutional data protection review, and appropriate retention/logging policy. Restrict Lambda concurrency and set AWS billing alarms to contain exposure. Do not rely on AI scores for summative grades. AI requests incur charges.

## Prerequisites

- GitHub account and repository
- AWS account, AWS CLI configured, AWS SAM CLI, Node.js/npm
- OpenAI API key with API billing enabled
- Permission from your institution to send student-written responses to the selected AI service

## 1. Store the API key in AWS Secrets Manager

In AWS Console → Secrets Manager → Store a new secret → Other type of secret. Use a plaintext secret containing your OpenAI API key (or JSON `{"OPENAI_API_KEY":"sk-..."}`). Name it `aws-storage-roleplay/openai`. Copy the **secret ARN**. Never commit the key to GitHub, JavaScript, or a CloudFormation parameter.

## 2. Deploy the backend

From repository root:

```bash
cd backend
npm install
cd ..
sam build
sam deploy --guided
```

Choose a stack name such as `aws-storage-roleplay`, a region, and supply:

- `OpenAISecretArn`: ARN from step 1
- `AllowedOrigin`: `https://YOUR_GITHUB_USERNAME.github.io` (origin only, **no repository path or trailing slash**)
- `AccessCode`: leave blank for initial private testing, or use a classroom code (not strong security)
- `OpenAIModel`: `gpt-4.1-mini` (change if necessary to an available model)

Approve IAM role creation. Note the `ChatEndpoint` output (ends with `/chat`). API Gateway uses HTTPS. Deployments may incur AWS and OpenAI charges.

## 3. Configure the GitHub Pages frontend

Edit `docs/config.js` and set `apiUrl` to the **ChatEndpoint**. If you supplied `AccessCode` during deployment, enter it here too (again: this is publicly readable and only deters accidental use).

Create a GitHub repository and push the project files. In GitHub repository → **Settings → Pages → Build and deployment → Deploy from a branch**; select `main` and `/docs`, then Save.

Your website should appear at:

`https://YOUR_GITHUB_USERNAME.github.io/YOUR_REPOSITORY_NAME/`

Use a private/incognito browser window to test start, reply, finish, transcript download, and mobile layout. For local frontend testing, use a local HTTP server; CORS will need a matching development origin if using the deployed API.

## 4. Blackboard Ultra instructions

1. Open the AWS course in Blackboard Ultra.
2. In **Course Content**, open Module 7 (Storage).
3. Click **+ → Create → Link** (names vary slightly by institution).
4. Name it `Module 7 – Meet Olivia: AWS Storage Role-Play`.
5. Paste the full GitHub Pages website URL.
6. Describe it as a 10-minute formative exercise; ask students to download their transcript and feedback if submission is required.
7. Make the item visible and test using Student Preview.

For embedding inside a Blackboard Document, ask your Blackboard administrator whether third-party iframe embeds are permitted. If allowed, the HTML is:

```html
<iframe title="AWS Storage Role-Play" src="https://YOUR_GITHUB_USERNAME.github.io/YOUR_REPOSITORY_NAME/" width="100%" height="850" style="border:0" loading="lazy"></iframe>
```

Iframe support depends on institutional policies and browsers; the **external link** is the most reliable method. This version does **not** integrate with Blackboard login or gradebook. That would require an LTI 1.3 integration and administrator registration.

## Classroom and technical notes

- The 10-minute timer begins when the first AI question is returned. The backend issues a signed expiry token and rejects new student replies after expiry (including requests that finish processing after expiry). The browser automatically ends the exercise and requests feedback at 00:00. Students may finish early. The server accepts assessment requests after expiry so that feedback can be generated. This is a time-limited formative exercise, not a tamper-proof proctored examination; a student can start a fresh session.
- If time expires, the student can request feedback based on the conversation so far.
- The AI aims to ask ~5 questions but cannot guarantee identical difficulty or coverage. Marking is approximate and formative.
- No automatic transcript upload or database storage. Downloaded files are controlled by the student.
- The frontend sends the entire conversation to the backend on each turn. The backend does not trust the transcript as proof of identity or work performed. Students could manipulate it; this is **not suitable for summative assessment**.
- API Gateway's CORS settings only control browser origins. They do not stop direct API calls. For larger deployments add proper authentication and rate controls.
- AWS CloudWatch may contain operational metadata and exceptions; configure retention and avoid logging student response content.
- Model output and grades can be inaccurate. Verify storage details against current AWS documentation.
- Avoid collecting personally identifiable or sensitive information; consult your institution's AI and data-protection policies before rollout.

## Test checklist

- [ ] GitHub Pages loads over HTTPS
- [ ] Backend endpoint works and CORS permits the GitHub Pages origin
- [ ] OpenAI key is stored only in Secrets Manager
- [ ] First question, reply and follow-up work
- [ ] Timer counts down; feedback is available
- [ ] Transcript downloads correctly
- [ ] Restart clears session
- [ ] Costs, concurrency and API abuse mitigations reviewed
- [ ] Student Preview in Blackboard works

## Mandatory signing secret

During `sam deploy --guided`, set `SessionSigningSecret` to a randomly generated value of at least 32 characters (for example, generate it locally with `python -c "import secrets; print(secrets.token_hex(32))"`). Keep this secret private. Changing it invalidates active sessions.

The hard limit is enforced by the Lambda server as well as the browser. At 00:00 the answer box is locked and assessment is automatically requested. If the AI request is in flight, the interface waits for it to settle before requesting assessment; the backend rejects late responses.
