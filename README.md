# Secure AgentCore Customer-Service Agent

A customer-service AI agent on AWS that can look up a customer's order — and can't be talked into showing anyone else's. Built and attacked in a personal lab to learn AI security hands-on.

**Result in one line:** I found that the AgentCore Gateway does not pass the user's identity to the tool, closed that gap with a request interceptor, and then tested two attacks against it.

## How I built this

**Why this exists.** Before this, I was doing AWS quests and labs by following the written instructions. I finished them, but I wasn't really getting the concepts. I wanted to talk through my problems and questions, and to have someone challenge what I believed. So I turned a quest into a portfolio: we argue, read the official documentation, check best practices, and learn from that.

I had no AWS AI project at work, so I designed my own case and used AI openly as my sparring partner — not to write the answer for me, but to argue with me.

- **It challenged every design.** I proposed, it pushed back, I defended or changed my mind — the way you debate options with a customer.
- **It gave me many of the commands.** I ran them, hit real errors (a quota of zero, a full disk, a package installed in the wrong folder) and fixed them myself.
- **It tested me afterwards.** I had to explain each part in my own words, and it corrected me when I was wrong.

My test for myself: can I explain the full token flow without AI? Yes. The debate moments — where I was wrong and where I held my ground — are written up in the portfolio document.

**The real lesson for me is the process of working with AI:** start with what you don't know, learn it by building, gain confidence, and keep improving. Then take on the next new thing and keep getting better. That is what I want to share — not a finished expert, but a way to keep learning.

Even this write-up was made the same way. I didn't give one command and sit in the corner — it was a conversation all the way through: I explained, the AI questioned, I corrected and decided.

## Architecture

Client (curl) → Cognito login (JWT) → **AgentCore Runtime** (JWT check) → Strands agent → **AgentCore Gateway** (JWT check + request interceptor) → Lambda tool → DynamoDB

- **Cognito** issues the user's token.
- **Runtime** validates the token before my code runs, and passes only the `Authorization` header through.
- **The agent** forwards the token unchanged to the Gateway. It never validates it and never caches it.
- **Gateway** validates the token again. Its **request interceptor** reads the user's identity from the token and writes it into the tool call, overwriting anything the model or caller sent.
- **Lambda** looks up the order by (user identity, order ID) and returns only the fields needed.
- **Model:** Claude Haiku via the Anthropic API. The key is stored in Secrets Manager; the Runtime role can read only that one secret.

## Key ideas

- **The model chooses the tool; the code enforces every decision.**
- **Trust is not passed, it is verified — gate by gate.**
- **The model never sees the key.** Identity travels in a header, not in the prompt.
- **One role, one job.** Least privilege at every layer.

## What I tested

| Test | What I did | Result |
| --- | --- | --- |
| Happy path | Sunny asks for his own order 123 | Status returned through the full chain |
| Confused deputy | Sunny asks for Ana's order 456 | "Not found" — same as a missing order, so nothing leaks |
| Indirect prompt injection | Order 789 had attack text planted in a free-text field | Model ignored it — but I removed the field anyway, because luck is not a control |

## What I learned

- The Gateway checks *who you are*, but it does not tell the tool. Identity has to be carried deliberately.
- Free-text fields written by other people are untrusted input, for the model and for the screen.
- A model behaving well today is not a security control. Removing data the model doesn't need is.
- What you build by clicking, you delete by clicking. Next time: infrastructure as code.

The full write-up — threat table, design decisions, build logs, debate moments and proof screenshots — is in the portfolio document.

## Status

Lab complete. All AWS resources and the API key were deleted after testing. No real account, user or customer data is included.
