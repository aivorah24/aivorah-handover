# Architecture

The full picture: the app, the 20 backend services grouped by area, and every data store and outside service they use. Built from the live Render settings and the code on 2026-11-03.

## System diagram

```mermaid
flowchart TB
    users["Students · Parents · Teachers · School admins · Aivorah admins"]
    fe["aivorah-frontend<br/>Expo web + mobile<br/>hosted on Vercel"]
    fba["Firebase Auth<br/>login"]
    users --> fe
    fe -- "sign in" --> fba
    fe -- "HTTPS + Firebase token" --> svcs

    subgraph svcs["20 backend services · Node/Express · Render"]
        direction TB
        subgraph acc["Accounts"]
            auth["auth"]
            profile["profile"]
            students["students"]
            pctl["parental-controls"]
        end
        subgraph learn["Learning"]
            lps["learning-paths"]
            asg["assignments"]
            coach["ai-coach"]
            chal["challenges"]
            lb["leaderboard"]
            tdash["teacher-dashboard"]
        end
        subgraph school["School"]
            sadmin["school-admin"]
            att["attendance"]
            ttr["timetable-resources"]
            sched["schedule"]
            msg["messages"]
        end
        subgraph extras["Extras"]
            habits["daily-habits"]
            stories["bedtime-stories"]
            wy["world-and-you"]
            help["help-center"]
        end
        padmin["platform-admin"]
    end

    sb[("Supabase Test DB<br/>Postgres + storage")]
    q[("Qdrant<br/>content vectors")]
    cld["Cloudinary<br/>images and files"]
    mail["Resend / Gmail<br/>email"]
    ai["AI providers<br/>Groq · Gemini · OpenRouter<br/>Mistral · Hugging Face · NVIDIA"]
    voice["ElevenLabs<br/>voice"]

    svcs --> sb
    lps --> q
    asg --> q
    coach --> q
    tdash --> q
    svcs --> cld
    svcs --> mail
    svcs --> ai
    lps --> voice
    stories --> voice
    msg -- "WebSocket + Expo push" --> fe
    students -- "invites" --> auth
    students --> lps

    mono["ai-mentor (legacy)<br/>aivorah.ai"]
    fe -. "older calls" .-> mono
    dbrepo["aivorah-db<br/>migrations"] -. "schema" .-> sb
    e2e["aivorah-e2e-tests<br/>Playwright"] -. "tests previews" .-> fe
```

## What each service uses

From each service's environment settings on Render.

| Service | Supabase | Firebase | Qdrant | Cloudinary | Email | AI and other |
|---|---|---|---|---|---|---|
| auth | ✅ | ✅ | | | Resend, Gmail | |
| profile | ✅ | | | ✅ | Resend | |
| students | ✅ | ✅ | | | Resend | Groq; calls auth, learning-paths |
| parental-controls | ✅ | | | | | |
| learning-paths | ✅ | ✅ | ✅ | ✅ | | Groq, Gemini, ElevenLabs |
| assignments | ✅ | | ✅ | ✅ | | Groq, Gemini, Mistral, Hugging Face |
| ai-coach | ✅ | | ✅ | | | Groq, NVIDIA |
| challenges | ✅ | | | | | |
| leaderboard | ✅ | | | | | |
| teacher-dashboard | ✅ | | ✅ | | | |
| school-admin | ✅ | ✅ | | ✅ | Resend, Gmail | Groq |
| attendance | ✅ | ✅ | | | | |
| timetable-resources | ✅ | ✅ | | ✅ | | Groq, Gemini, OpenRouter |
| schedule | ✅ | ✅ | | | | |
| messages | ✅ | ✅ | | ✅ | | Groq; WebSocket, Expo push |
| daily-habits | ✅ | | | | | |
| bedtime-stories | | | | | | Groq, ElevenLabs |
| world-and-you | ✅ | | | | | Groq, News API |
| help-center | | | | | Gmail | Groq |
| platform-admin | ✅ | ✅ | | | Resend, Gmail | Gemini; calls auth |

Service addresses: [../repositories.md](../repositories.md). Which screens use which service: [FEATURE_SERVICE_MAP.md](FEATURE_SERVICE_MAP.md).
