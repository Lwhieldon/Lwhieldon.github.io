---
layout: post
title: "🔐 Post-Mortem: The Auth Failure That Broke My Foundry Demo"
date: 2026-06-10
author: Lee Whieldon
description: My Microsoft Foundry demo died on a credential chain that hung for fifty seconds waiting on an Azure metadata endpoint that was never going to respond. Here is what really failed, why DefaultAzureCredential is a time bomb on local dev machines, and the four-part fix I shipped next.
---

## 🎬 What Actually Happened

Today at 12:00 PM ET I went live on the Microsoft Reactor stage to demo [**Compliance Academy**](https://github.com/lwhieldon/msft-enterprise-learning-agent) at the Reasoning Agents Battle. Three weeks of evening builds, four agents, fifty-two policy chunks indexed against Azure AI Search, a live activity log streaming next to the player UI. The whole thing rehearsed end-to-end. The whole thing working flawlessly twenty minutes before going live.

When it was time for the live demo segment, the part of the format where the hosts wanted to see the product actually working (agents reasoning in real time, retrieval grounding citations, the whole engine doing what the previous segments described), I clicked on a suspect, asked a question, and watched the Forensic Analyst spinner spin. And spin. And keep spinning.

The activity log told the real story. Or rather, the Chainlit log did, once I went back through it after going off-air:

```
12:42:25 - INFO - azure.identity - No environment configuration found.
12:42:25 - INFO - azure.identity - ManagedIdentityCredential will use IMDS
12:42:31 - INFO - azure.search - Request URL: '.../search.post.search?api-version=2024-07-01'
12:42:34 - INFO - azure.search - Response status: 200
12:42:43 - INFO - azure.identity - GET http://169.254.169.254/metadata/identity/oauth2/token
12:43:32 - WARNING - azure.identity - DefaultAzureCredential failed to retrieve a token
        EnvironmentCredential: ...unavailable. Environment variables are not fully configured.
        ManagedIdentityCredential: ...unavailable, no response from the IMDS endpoint.
        AzureCliCredential: Failed to invoke the Azure CLI
        AzurePowerShellCredential: Az.Account module >= 2.2.0 is not installed
        AzureDeveloperCliCredential: Azure Developer CLI could not be found.
```

Two things jump out. **The search call succeeded.** That request to the Azure AI Search index returned 200, because the search client in my codebase has an API key fallback when one is set in `.env`. Retrieval was fine. The compliance content was available.

**The Azure OpenAI call never made it out the door.** Or more accurately, the credential acquisition that was supposed to authorize the model call never completed. The Forensic Analyst was about to call the model to ground its response in the retrieved policy chunks. That call goes through `DefaultAzureCredential`, which has **no API key fallback** configured. The credential walked through every option it knew: Environment, Workload Identity, Managed Identity, Shared Token Cache, Visual Studio, Azure CLI, Azure PowerShell, Azure Developer CLI, Broker. Every single one failed. The decisive line is `AzureCliCredential: Failed to invoke the Azure CLI`. That was the one I had been relying on.

I had about ninety seconds of stream time to recover. I tried to refresh by running `az login` in another terminal. That command failed too, with a different error that I will get to in a moment.

I could not recover. The demo ended without showing inference.

This post is the diagnosis and the fix. If you are building live demos on Microsoft Foundry, or any service that authenticates through Microsoft Entra ID, do not make the same mistake I did. The TL;DR: **`DefaultAzureCredential` is wrong for any workload running on a developer laptop that needs to keep running without a human at the keyboard.**

---

## 🧪 The Setup That Failed

The auth code in question is the one I had been showing off just minutes earlier during the code walkthrough segment:

```python
def build_azure_client():
    """Construct an AzureOpenAI client via Entra ID."""
    project_endpoint = require_env("AZURE_AI_PROJECT_ENDPOINT")
    resource_endpoint = (
        project_endpoint.split("/api/projects/")[0].rstrip("/") + "/"
    )

    token_provider = get_bearer_token_provider(
        DefaultAzureCredential(),
        "https://cognitiveservices.azure.com/.default",
    )

    return AzureOpenAI(
        azure_endpoint=resource_endpoint,
        azure_ad_token_provider=token_provider,
        api_version=os.environ.get(
            "AZURE_OPENAI_API_VERSION", DEFAULT_API_VERSION
        ),
    )
```

Minutes earlier, the hosts had asked each of us to walk through our code and explain how the agents were actually built. They were trying to surface the implementation choices behind each project, not just the polished UI. I had narrated this exact block as the *"enterprise-grade auth becomes the default, not an upgrade"* moment. No API keys, no secret rotation, no `.env` file with sensitive material. The same `az login` that authorizes the Azure CLI authorizes every agent call. Clean. Beautiful. And, as it turned out, fragile in exactly the wrong way for a live demo.

---

## 🔎 What Actually Broke (And Why)

The auth failure has three layers stacked on top of each other. Each one is its own lesson.

### 1. The Azure CLI session was not valid on the demo machine

The decisive log line is `AzureCliCredential: Failed to invoke the Azure CLI`. Whether `az` itself was missing from the PATH at runtime, or the cached login token had expired, or some other interaction with the corporate environment had unsettled the CLI state, the practical reality was the same: when my agent went to acquire a token, the credential method I had wired up could not produce one.

The deeper issue is that I had treated `az login` as a one-time setup step rather than a continuously-verified state. The CLI session can fail in ways the application has no visibility into until the next call goes out. By the time the demo segment started, the application had no idea whether auth would work. It just trusted that I had run `az login` correctly during pre-flight. The pre-flight check verified I could run the Azure CLI in a terminal; it did not verify that the SDK could successfully invoke it from within Python at request time.

### 2. `DefaultAzureCredential` hangs for fifty seconds waiting on Managed Identity

Here is the part that turned a failed credential into a dead demo. When `AzureCliCredential` failed, `DefaultAzureCredential` did not error out fast. It moved on to the next credential type in its chain. That chain includes `ManagedIdentityCredential`, which tries to fetch a token from the Azure Instance Metadata Service at `http://169.254.169.254`.

That endpoint only exists inside Azure. On a corporate Windows laptop running outside Azure, the request to `169.254.169.254` just sits there, waiting for a response that will never come. The SDK default timeout for this path is roughly fifty seconds. **The audience saw a spinner for fifty seconds because `DefaultAzureCredential` was probing an Azure metadata service that was never going to respond.**

This is the surprise lesson from today. `DefaultAzureCredential` is sold as a convenience: same code in dev, in production, in CI. What no one tells you up front is that the convenience comes with a fifty-second time bomb embedded in the local-dev path, sitting at the exact place where you would notice it least: when something else has already gone wrong and you are panicking.

### 3. The recovery path (`az login`) failed because of corporate TLS interception

In the moment, I tried to recover by running `az login` in a different terminal. That command failed too, but with a different error:

```
HTTPSConnectionPool(host='login.microsoftonline.com', port=443):
  Max retries exceeded with url: /organizations/v2.0/.well-known/openid-configuration
Caused by SSLError(SSLCertVerificationError(
  1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed:
  self-signed certificate in certificate chain'))
```

This is a separate problem from the credential chain. The corporate laptop runs Netskope, a TLS-intercepting proxy that re-signs HTTPS traffic with its own root certificate. Browsers trust this re-signing because Windows already has the corporate root cert installed. The Azure CLI does not. It ships its own Python interpreter and its own CA bundle, neither of which trust the corporate root.

I had worked around this earlier in the morning by setting `REQUESTS_CA_BUNDLE` to point at the corporate cert bundle (`C:\Users\lwhieldon\corp-ca-bundle.pem`), but I had set it for the current PowerShell session only, not persisted to user environment. When the demo session started, the variable was gone. Recovery via `az login` was not possible without first re-setting that variable, which I had no time or composure to do live on stream.

Three failure layers stacking on each other. Each one was survivable on its own. All three together, in front of a live audience, on the agent call I most needed to work: not survivable.

---

## 🛠️ The Fix: Four Layers

The architectural lesson is straightforward. The mechanism I used was right for interactive development. It was wrong for a non-interactive workload (a streaming demo where I cannot pause to refresh credentials). Those are different jobs. They need different auth. And the abstraction I was using assumed sameness it could not deliver.

Here is the four-part fix I am shipping to the [main repo](https://github.com/lwhieldon/msft-enterprise-learning-agent). Each layer addresses one of the failure modes from the previous section.

### Fix 1: Replace `DefaultAzureCredential` with an explicit, short chain

The first fix kills the fifty-second IMDS time bomb. Instead of using `DefaultAzureCredential` (which silently walks ten credential types including `ManagedIdentityCredential`), I am building an explicit `ChainedTokenCredential` that only includes credentials that make sense for this workload:

```python
def build_credential():
    """Build a token credential with stream-day-safe fallback.

    Tries credentials in this order:
      1. Service Principal (if AZURE_CLIENT_ID, AZURE_CLIENT_SECRET,
         AZURE_TENANT_ID are all set). Persistent, non-interactive,
         no fifty-second IMDS hang. Use this for live demos.
      2. AzureCliCredential (requires `az login`). Developer-friendly
         default for daily local work.

    DELIBERATELY OMITTED: ManagedIdentityCredential. This app does not
    run inside Azure. Including it means a fifty-second hang every time
    the other credentials fail.
    """
    from azure.identity import (
        AzureCliCredential,
        ClientSecretCredential,
        ChainedTokenCredential,
    )

    credentials = []

    client_id = os.environ.get("AZURE_CLIENT_ID", "").strip()
    client_secret = os.environ.get("AZURE_CLIENT_SECRET", "").strip()
    tenant_id = os.environ.get("AZURE_TENANT_ID", "").strip()

    if client_id and client_secret and tenant_id:
        credentials.append(ClientSecretCredential(
            tenant_id=tenant_id,
            client_id=client_id,
            client_secret=client_secret,
        ))

    credentials.append(AzureCliCredential())

    if len(credentials) == 1:
        return credentials[0]
    return ChainedTokenCredential(*credentials)
```

The chain only contains `ClientSecretCredential` (if SP env vars set) and `AzureCliCredential` as fallback. **`ManagedIdentityCredential` is intentionally omitted.** If both fail, the chain fails immediately. No fifty-second IMDS probe. No audience-visible hang. Just a clean error the application can report.

### Fix 2: Add a Service Principal for live demos

A Service Principal is a non-human identity in Microsoft Entra ID. It authenticates with a tenant ID + client ID + client secret. It exchanges those for a token via the OAuth2 client credentials grant, and the refresh path is **non-interactive and silent**. There is no "your session has expired" failure. There is no dependency on a working Azure CLI on the demo machine. The whole class of "my CLI session died and I have no way to know it died" failures goes away.

You create one like this:

```powershell
az ad sp create-for-rbac `
  --name "compliance-academy-stream" `
  --role "Cognitive Services OpenAI User" `
  --scopes "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<foundry-resource>"
```

This returns `appId`, `password`, and `tenant`. Those go into your local `.env` file (which is in `.gitignore`, so they stay off GitHub). You also grant the SP `Search Index Data Reader` on the Azure AI Search service so the retrieval calls still work via Entra ID:

```powershell
az role assignment create `
  --assignee <appId> `
  --role "Search Index Data Reader" `
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Search/searchServices/<search>"
```

Once wired up, `build_credential()` from Fix 1 picks up the SP automatically. The codebase does not need to change. Daily dev still uses `az login` (no SP env vars set, fallback to `AzureCliCredential`). Stream day uses the SP (env vars populated from `.env`). Same code path, different credentials.

### Fix 3: Add an API key fallback to the Azure OpenAI client

This one was hiding in plain sight, and finding it stung. The search client already has an API key fallback. That is why the search call succeeded in the failed demo even with Entra ID completely broken. The Azure OpenAI client did not have the same fallback. Adding one mirrors the existing pattern:

```python
def build_azure_client():
    """Construct an AzureOpenAI client.

    Tries Entra ID via build_credential() first. Falls back to API
    key from AZURE_OPENAI_API_KEY if Entra ID is unavailable. This
    matches the dual-path pattern already used by _search_client.py.
    """
    from openai import AzureOpenAI

    project_endpoint = require_env("AZURE_AI_PROJECT_ENDPOINT")
    resource_endpoint = (
        project_endpoint.split("/api/projects/")[0].rstrip("/") + "/"
    )
    api_version = os.environ.get("AZURE_OPENAI_API_VERSION", DEFAULT_API_VERSION)

    api_key = os.environ.get("AZURE_OPENAI_API_KEY", "").strip()
    if api_key:
        return AzureOpenAI(
            azure_endpoint=resource_endpoint,
            api_key=api_key,
            api_version=api_version,
        )

    from azure.identity import get_bearer_token_provider
    token_provider = get_bearer_token_provider(
        build_credential(),
        "https://cognitiveservices.azure.com/.default",
    )
    return AzureOpenAI(
        azure_endpoint=resource_endpoint,
        azure_ad_token_provider=token_provider,
        api_version=api_version,
    )
```

For daily dev: stay on Entra ID, no key needed in `.env`. For live demos: keep the key available in `.env` as the last-resort fallback. The "no API keys in code" narrative survives intact because the key only lives in `.env` (gitignored). What it does NOT survive is the "no API keys anywhere" narrative, which I had told myself was sustainable for a corporate-laptop demo and which today's failure proved was not.

The tradeoff is honest. Pure Entra ID auth is the right default. A keyed fallback is the right insurance policy. Both can coexist.

### Fix 4: Pre-flight that verifies SDK-level token acquisition, not just CLI exit code

The other thing that bit me was treating `az login` success in PowerShell as proof that the application would be able to acquire a token. Those are different things. The CLI returning exit code 0 means the CLI thinks it logged in. The SDK invoking the CLI from within Python at request time is a different code path with different failure modes.

The fix is a `warm_up_auth()` function that **actually fetches a token via the SDK** during pre-flight:

```python
def warm_up_auth() -> dict:
    """Force a fresh token acquisition through the same code path the
    app uses at runtime. Use in pre-flight to verify auth ACTUALLY
    works AND prime the token cache so the first agent call pays no
    auth latency cost.

    Raises if auth fails, with the resolved credential type in the
    error message so you know exactly which path was attempted.
    """
    credential = build_credential()
    try:
        token = credential.get_token(
            "https://cognitiveservices.azure.com/.default"
        )
    except Exception as exc:
        raise AgentClientError(
            f"Pre-flight auth check failed via "
            f"{type(credential).__name__}: {exc}"
        ) from exc

    return {
        "credential_type": type(credential).__name__,
        "expires_on": token.expires_on,
    }
```

New pre-flight runs this check before Chainlit accepts requests:

```powershell
python -c "from src.agents._azure_client import warm_up_auth; print(warm_up_auth())"
# {'credential_type': 'ClientSecretCredential', 'expires_on': 1718045833}
```

If the credential type comes back as `ClientSecretCredential`, the Service Principal is wired up. If it comes back as `AzureCliCredential`, you are about to run a demo on the same code path that broke for me today. Decide accordingly. If it throws, fix it now, not at 12:50.

---

## 🎚️ Auth Health in the Activity Log

The other thing I am adding for the next demo is an explicit auth health emission to the activity log. Not just whether a token is valid, but **which credential type actually resolved**. That is the metric that would have warned me today:

```python
def _emit_auth_health(credential_type: str, token_expires_on: int) -> None:
    """Emit auth health to the activity log on startup and periodically."""
    now = int(time.time())
    minutes_left = max(0, (token_expires_on - now) // 60)
    _emit(
        "Auth",
        f"Active credential: {credential_type} "
        f"(token valid for {minutes_left} more minutes)",
    )
```

If the activity log had been emitting `Active credential: AzureCliCredential` during the code walkthrough, I would have seen that line as I narrated and known the demo was on a fragile path. If I had been on the Service Principal, the log would say `Active credential: ClientSecretCredential` and I would have known I was safe. **Observability on auth is observability on the most failure-prone thing in the system.** Treat it as a first-class concern.

---

## 🧠 The Underlying Lesson

The bigger takeaway, the one I want to make sure I do not lose: **the auth pattern needs to match the workload, not the developer's daily habit.**

| Workload | Right auth |
| --- | --- |
| Local development | `AzureCliCredential` via `az login`. Interactive, scoped to your user, ties to your existing SSO session. |
| CI / Build pipelines | Service Principal via federated identity or short-lived secret. Non-interactive, runs without a human. |
| **Live demos** | **Service Principal.** Same logic as CI. There is no human available to re-authenticate mid-stream. |
| Production services | Managed Identity. The cloud platform manages the credential lifecycle for you. |
| Multi-tenant apps | App registration with delegated permissions and proper consent flows. |

`DefaultAzureCredential` is appealing because it papers over the difference. The same code "just works" in all five contexts. But "just works" in this case means "tries a bunch of stuff in sequence and hopes." When something does fail (and on a corporate-proxy laptop running a live demo, something will eventually fail), the abstraction is too thick to debug in 90 seconds of stream time.

The discipline I am keeping going forward is: **be explicit about which credential the code is supposed to use for the workload it is in, and surface that decision in startup logs so future-me knows what assumptions are in play.**

---

## 🎙️ What I Wish I Had Done Differently

If I could rewind to my T-30 pre-flight:

1. **Set up a Service Principal for the demo profile, not just `az login`.** Five-minute one-time setup. Removes the entire failure mode where the CLI session has silently become unusable from inside Python by the time the demo starts.
2. **Persist `REQUESTS_CA_BUNDLE` to user environment variables, not just the current PowerShell session.** That way even if I need to fall back to `az login` mid-demo as recovery, the CLI's bundled Python actually trusts the corporate root cert. Five-second setup. Eliminates the recovery-path failure that compounded today's auth failure.
3. **Run a real `warm_up_auth()` check in pre-flight that fetches a token via the SDK, not just confirms `az login` exited cleanly.** Earlier in the day, `az login` worked in my PowerShell. By the time the demo needed auth, the SDK could not invoke the CLI from inside Python. The CLI and the SDK are different code paths with different failure modes, and my pre-flight only exercised one of them.
4. **Wire the auth health emission into the activity log from the start.** If the log had been showing `Active credential: AzureCliCredential` during the code-walkthrough segment minutes before the demo, the warning would have been visible on screen, in the room, and in the moment.

The thing about live demos is that **the auth path is the only part you cannot script around in the moment**. Every other failure (a slow generate, a search returning nothing, a UI glitch) has a fallback or a narration. Auth failing means inference is impossible, and inference is the whole point of an AI demo.

---

## 🚀 What's Next for Compliance Academy

Despite today's auth failure, the rest of the build held up. The architecture is solid. The retrieval grounding is working. The scenario generator hot-loads new cases reliably. I am genuinely proud of what the engine does once it can actually call the model.

So the immediate next chunk of work:

- **Replace `DefaultAzureCredential` with the explicit short chain** from Fix 1, removing the IMDS hang from the local-dev path entirely.
- **Add the API key fallback to the Azure OpenAI client** from Fix 3, mirroring the dual-path pattern the search client already has.
- **Ship the Service Principal credential path** so live demos can run on a non-interactive identity that does not depend on a working Azure CLI.
- **Ship the `warm_up_auth()` pre-flight check** from Fix 4 and wire it into Chainlit startup so any auth issue surfaces before the UI even comes up, not when the first agent call fires.
- **Add the auth health emission** to the activity log so the demo surface includes auth visibility by default.
- **Document the demo-machine setup**, including persistent `REQUESTS_CA_BUNDLE` configuration and SP env var setup, in `docs/live_demo_setup.md` so anyone forking the repo for their own demo can avoid this exact failure mode.
- **Re-run the demo** at a future opportunity, either on a smaller stream or as a recorded walkthrough, so the work I put into this gets a fair showing.

If you are working on your own live demo and want to compare auth-hardening notes, find me on [LinkedIn](https://www.linkedin.com/in/lee-w-9b3a0620/) or [GitHub](https://github.com/Lwhieldon). And if you also had a live-demo failure that taught you something, I want to hear about it. The engineering community gets better when we share these.

Thanks to [Lee Stott](https://www.linkedin.com/in/leestott/) and [Carlotta Castelluccio](https://www.linkedin.com/in/carlotta-castelluccio/) for hosting the Reactor battle and giving these projects a stage, even when they do not go to plan. 🎤

---

*This site is open source. [Improve this page](https://github.com/Lwhieldon/Lwhieldon.github.io/edit/main/_posts/2026-06-10-foundry-auth-timeout-postmortem.md).*
