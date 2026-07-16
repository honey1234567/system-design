I'll explain Netflix system design as if you're a beginner preparing for interviews.

https://systemdesignschool.io/problems/netflix/solution?utm_source=neetcode

## First: What is the actual problem Netflix is solving?

Imagine:

* 300M+ users
* Millions watching simultaneously
* Movies are huge (2GB–50GB)
* Users expect:

  * Video starts in <2 sec
  * No buffering
  * Resume from last position
  * Personalized recommendations

The biggest challenge is:

> How do we deliver huge video files to millions of users worldwide without buffering?

Unlike YouTube, Netflix is mainly a **video delivery problem**, not a video upload problem. ([Sujeet Jaiswal][1])

---

# Beginner's High Level Architecture

```text
User App
    │
Load Balancer
    │
API Gateway
    │
──────────────────────────
│ Auth Service           │
│ Recommendation Service │
│ Search Service         │
│ Playback Service       │
│ Watch History Service  │
──────────────────────────
    │
Metadata DB
    │
CDN
    │
User Streams Video
```

---

# Component 1: Metadata vs Video Storage

Beginners often think database stores movies.

Wrong.

### Metadata DB stores

```json
{
  "movieId": 100,
  "title": "Money Heist",
  "genre": "Crime",
  "duration": "45 mins",
  "videoUrl": "cdn.netflix.com/movie100"
}
```

Only information about movie.

### Video Storage stores

Actual movie:

```text
MoneyHeist.mp4
```

maybe 5GB.

Usually stored in Object Storage. ([System Design School][2])

---

# Component 2: Why CDN?

Suppose Netflix stores everything in US.

You are watching from India.

```text
India User
      │
      ▼
USA Server
```

Every video chunk travels across continents.

Result:

❌ High latency

❌ Buffering

❌ Huge bandwidth cost

---

### Solution: CDN

CDN = Content Delivery Network

Netflix copies videos closer to users.

```text
USA Storage
      │
 ┌────┼────┐
 ▼    ▼    ▼

India Europe Japan

 CDN   CDN   CDN
```

Indian user gets movie from India CDN.

Much faster. ([System Design School][2])

---

# Real Netflix Secret: Open Connect

Most companies use third-party CDN.

Netflix built its own CDN called Open Connect.

Netflix places servers inside ISP networks.

```text
You
 │
Jio ISP
 │
Netflix Server
```

Movie travels very short distance.

This is one of Netflix's biggest innovations. ([System Design School][2])

---

# Component 3: Video Encoding

Suppose original movie:

```text
4K
50 GB
```

Can every user stream it?

No.

Someone may have:

* Slow 3G
* Mobile phone
* Laptop
* Smart TV

---

Netflix creates multiple versions.

```text
Movie

├── 240p
├── 480p
├── 720p
├── 1080p
├── 4K
```

This process is called Encoding/Transcoding. ([SysDesignWiki][3])

---

# Component 4: Adaptive Bitrate Streaming

This is one of the most important interview concepts.

Imagine:

```text
Current Speed = 50 Mbps
```

Netflix serves:

```text
1080p
```

Suddenly internet becomes:

```text
5 Mbps
```

Instead of buffering:

Netflix switches to:

```text
720p
```

or

```text
480p
```

automatically.

User sees slightly lower quality but no buffering.

This is called Adaptive Bitrate Streaming. ([Educative][4])

---

# Component 5: Video Chunking

Netflix never sends whole movie.

Bad:

```text
Movie = 5 GB

Send Entire File
```

User waits forever.

---

Instead:

```text
Movie

Chunk1
Chunk2
Chunk3
Chunk4
Chunk5
```

Each chunk may represent a few seconds.

Flow:

```text
Play Clicked

Download Chunk1

Start Playing

Download Chunk2

Download Chunk3
```

Video starts almost immediately.

---

# Component 6: Playback Service

When you press Play:

```text
User
 │
Play Request
 │
Playback Service
```

Playback Service decides:

* Which CDN?
* Which video quality?
* Which device?
* DRM rules?
* Resume position?

Then returns streaming URL. ([SysDesignWiki][3])

---

# Component 7: Watch History

Suppose you stop at:

```text
Episode 4
Minute 23:45
```

Stored as:

```json
{
 "userId":1,
 "movieId":10,
 "position":"23:45"
}
```

Next login:

```text
Continue Watching
```

appears.

---

# Component 8: Recommendation System

This is another huge Netflix topic.

Netflix tracks:

* Watch history
* Completion %
* Rewatch
* Language
* Device
* Time of day

Example:

You watched:

```text
Money Heist
Narcos
Breaking Bad
```

Netflix predicts:

```text
Ozark
```

You may like it too. ([Netflix Help Center][5])

---

## Simplified Recommendation Flow

```text
User Activity
      │
Event Stream
      │
ML Models
      │
Recommended Movies
```

Interviewers love this section.

---

# Component 9: Search

When user types:

```text
spi
```

Netflix shows:

```text
Spider-Man
Spirited Away
```

Search system usually uses:

* Elasticsearch/OpenSearch
* Inverted Index

---

# Component 10: Cache

Without cache:

```text
DB hit for every request
```

Millions of users = DB dies.

Use Redis:

```text
User
 │
Redis Cache
 │
Database
```

Popular data served from cache.

---

# Complete Playback Flow

Suppose Shreya clicks "Money Heist".

```text
1 User clicks Play

2 API Gateway

3 Playback Service

4 Metadata DB

5 Get Video URL

6 Nearest CDN Selected

7 Chunk 1 Sent

8 Video Starts

9 More Chunks Downloaded

10 Watch History Updated
```

This is the most important sequence diagram to remember.

---

# Netflix vs YouTube (Interview View)

### YouTube Focus

```text
Upload
Encoding
Copyright
Comments
Likes
Subscriptions
```

### Netflix Focus

```text
CDN
Streaming
Adaptive Bitrate
Recommendations
Subscription
Watch History
```

---

# 5 Things Interviewers Expect You To Mention

If asked "Design Netflix", always cover:

### 1. CDN

Most important.

### 2. Video Encoding

Multiple resolutions.

### 3. Adaptive Bitrate Streaming

Quality changes automatically.

### 4. Recommendation Engine

Personalization.

### 5. Watch History

Resume playback.

These 5 topics alone cover most of the scoring points in a Netflix system design interview. ([System Design School][2])

### One-line summary

**YouTube = "How do users upload and share videos?"**

**Netflix = "How do we deliver pre-recorded videos to millions of users worldwide with zero buffering and personalized recommendations?"** ([Sujeet Jaiswal][1])

[1]: https://sujeet.pro/articles/system-design/system-design-problems/design-netflix-streaming.html?utm_source=chatgpt.com "Design Netflix Video Streaming — Sujeet Jaiswal - Principal Software Engineer"
[2]: https://systemdesignschool.io/problems/netflix/solution?utm_source=chatgpt.com "Design Netflix: A Comprehensive Guide"
[3]: https://sysdesign.wiki/systems/netflix/?utm_source=chatgpt.com "Netflix - System Design Case Study"
[4]: https://www.educative.io/blog/ott-system-design?utm_source=chatgpt.com "OTT System Design Explained"
[5]: https://help.netflix.com/en/node/100639?utm_source=chatgpt.com "How Netflix’s Recommendations System Works | Netflix Help Center"

This is one of the **most important concepts** in Netflix system design, and many beginners misunderstand it.

Your question is essentially:

> **"YouTube also streams videos, so why is Netflix famous for streaming? And how does putting servers inside an ISP help?"**

Let's build the understanding from scratch.

---

# Part 1: What happens when you watch a video?

Suppose you're in Delhi and open Netflix.

You click **Money Heist**.

Without any optimization, the flow looks like this:

```text
You (Delhi)
      │
      │ Internet
      ▼
Netflix Server (USA)
```

Your movie has to travel thousands of kilometers.

That creates problems:

* Higher latency
* More internet congestion
* Higher bandwidth costs
* Buffering during peak hours

Even if your internet speed is 100 Mbps, the **distance and congestion** still matter.

Think of ordering food from:

* A restaurant 500 meters away
* A restaurant 30 km away

Even if both cook equally fast, the nearby one usually reaches you much sooner.

---

# Part 2: What is an ISP?

ISP = Internet Service Provider.

Examples in India:

* Jio Fiber
* Airtel Xstream
* ACT
* BSNL

When your phone requests a Netflix movie:

```text
Phone

↓

WiFi Router

↓

Jio Fiber

↓

Internet

↓

Netflix Server
```

Normally, Jio forwards your request across the internet until it reaches Netflix.

---

# Part 3: What Netflix changed

Netflix thought:

> Why send every movie across the internet if the same movie is watched millions of times?

Instead of keeping movies only in its own data centers, Netflix installs its own cache servers **inside or directly connected to ISP networks**. This system is called **Open Connect**.

Now the path becomes:

```text
Your Phone
      │
WiFi Router
      │
Jio Network
      │
Netflix Open Connect Server
```

Notice something?

The request **never needs to leave Jio's network** for popular content.

---

# Real-life analogy

Imagine Amazon.

### Old way

```
Amazon Warehouse (Mumbai)

↓

Courier

↓

Delhi
```

Every package travels from Mumbai.

---

### Better way

Amazon builds warehouses in Delhi.

```
Delhi Warehouse

↓

Courier

↓

Your Home
```

Delivery is much faster.

Netflix does exactly this for movies.

Instead of moving warehouses, it moves **video servers**.

---

# What is stored on Open Connect?

Not the entire Netflix catalog.

Typically:

* Trending movies
* Popular TV shows
* Frequently watched content
* Multiple quality versions (480p, 720p, 1080p, 4K)

Less popular content can still be fetched from regional or central data centers.

---

# Part 4: Why does this reduce buffering?

Suppose a movie is 6 GB.

### Without Open Connect

```
USA

↓

Atlantic Ocean

↓

Europe

↓

Middle East

↓

India

↓

Jio

↓

Your Phone
```

Many routers and networks are involved.

Any congestion along the way can slow the stream.

---

### With Open Connect

```
Jio Network

↓

Netflix Cache

↓

Your Phone
```

Much shorter path.

Lower latency.

Less congestion.

Less chance of buffering.

---

# Part 5: Why ISPs like this

Imagine 10 million Jio users watching Netflix at night.

Without local caching:

```
10 million requests

↓

International Internet Links
```

Huge bandwidth usage.

Very expensive.

---

With Open Connect:

```
10 million users

↓

Jio Local Netflix Server
```

Traffic stays mostly inside Jio's own network.

Benefits:

* Lower international bandwidth costs
* Better customer experience
* Less network congestion

So both Netflix **and** the ISP benefit.

---

# Part 6: But YouTube also streams videos. Why isn't it the same?

Excellent question.

Both stream videos, but their priorities are different.

## YouTube

Anyone can upload a video.

Every minute, hundreds of hours of new videos are uploaded.

That means YouTube constantly handles:

* Uploads
* Virus scanning
* Copyright detection
* Thumbnail generation
* Encoding
* Recommendations
* Comments
* Likes

Streaming is important, but the platform is also an enormous **content ingestion system**.

---

## Netflix

Only Netflix (or licensed studios) uploads content.

Maybe a few thousand new titles are added over time—not millions of user-generated uploads every day.

Netflix doesn't need to solve:

* Public uploads
* Copyright detection for user uploads
* Comments
* Likes

Instead, almost all engineering effort goes into **delivering video perfectly**.

---

# Why Netflix focuses more on streaming

Imagine:

## YouTube

```
100 million videos

Many videos watched once

Some watched millions of times
```

The "long tail" of rarely watched content is huge.

---

## Netflix

```
Top 20 shows

Watched by millions

Again and again
```

Shows like *Stranger Things* or *Wednesday* generate massive concurrent demand.

This makes aggressive caching extremely effective.

---

# Part 7: Adaptive Bitrate Streaming

Suppose you start watching in 4K.

```
Internet Speed

100 Mbps
```

Netflix sends:

```
4K
```

Suddenly your speed drops:

```
5 Mbps
```

Instead of stopping playback:

```
4K

↓

1080p

↓

720p

↓

480p
```

The switch happens seamlessly, usually at the next video segment.

That's why you often don't notice the quality change.

---

# Part 8: Chunk-based Streaming

Netflix never sends a whole 5 GB movie at once.

Instead:

```
Movie

↓

Chunk 1 (2–6 seconds)

Chunk 2

Chunk 3

Chunk 4
```

The player requests one chunk after another.

Benefits:

* Fast startup
* Easy quality switching between chunks
* Efficient retries if one chunk fails
* Resume playback from a specific point

---

# Part 9: Isn't YouTube doing adaptive streaming too?

Yes.

Both YouTube and Netflix use technologies such as MPEG-DASH or HLS to stream videos in chunks with multiple quality levels.

The **technology is similar**.

The difference is **what each company optimizes for**.

| YouTube                             | Netflix                                                  |
| ----------------------------------- | -------------------------------------------------------- |
| Upload millions of new videos daily | Deliver licensed content flawlessly                      |
| Comments, likes, creators           | Personalized viewing experience                          |
| Viral, unpredictable content        | High-quality streaming of popular titles                 |
| Many rarely viewed videos           | High concurrent viewers for the same content             |
| Global CDN                          | Global CDN + Open Connect appliances inside ISP networks |

---

# Part 10: Interview Answer

If an interviewer asks:

> **"Why is Netflix system design different from YouTube?"**

A strong beginner-friendly answer is:

> Both use CDNs, adaptive bitrate streaming, chunk-based delivery, and multiple encoded video qualities. The main difference is that YouTube must handle massive user-generated uploads and social features, while Netflix focuses on delivering a smaller catalog of licensed content with extremely high reliability. Netflix further improves performance through **Open Connect**, where it deploys cache servers inside or very close to ISP networks. This reduces the distance data travels, lowers ISP bandwidth costs, reduces latency and congestion, and helps millions of users stream with minimal buffering.

---

## A simple way to remember it

Think of the two platforms like logistics companies:

* **YouTube** is like a marketplace where millions of sellers upload new products every day. The biggest challenge is accepting, processing, organizing, and making those products discoverable.
* **Netflix** is like a premium retail chain with a fixed catalog. The biggest challenge is ensuring that every customer receives the product quickly and reliably. It achieves this by stocking popular "products" (videos) in warehouses (Open Connect servers) located inside or very close to local delivery networks (ISPs). This is why Netflix is often cited as the benchmark for large-scale video streaming system design.

This is one of the most common interview questions. The confusion comes because **both YouTube and Netflix stream videos using similar technologies** (CDN, HLS/DASH, adaptive bitrate, chunks). The real difference is **what they optimize for**.

Let's compare them step by step.

---

# High-Level Difference

| YouTube                               | Netflix                              |
| ------------------------------------- | ------------------------------------ |
| Upload-first platform                 | Streaming-first platform             |
| Billions of user-generated videos     | Licensed, curated content            |
| Biggest challenge: processing uploads | Biggest challenge: delivering videos |
| Unpredictable traffic                 | Predictable popular content          |
| Creator ecosystem                     | Subscription ecosystem               |

Think of it like this:

* **YouTube:** "How do I allow anyone in the world to upload videos?"
* **Netflix:** "How do I deliver movies to millions with zero buffering?"

---

# 1. Where do videos come from?

## YouTube

Anyone can upload.

```text
Creator

↓

Upload Video

↓

YouTube
```

Every minute, hundreds of hours of new video are uploaded.

Examples:

* Cooking videos
* Gaming
* College lectures
* Shorts
* Music
* Tutorials

Every upload must be processed.

---

## Netflix

Only Netflix or licensed studios upload.

```text
Movie Studio

↓

Netflix Content Team

↓

Netflix
```

New uploads are relatively infrequent.

Instead of billions of videos, Netflix focuses on a much smaller catalog.

---

# 2. Upload Pipeline

## YouTube

The upload pipeline is huge.

```text
Creator

↓

Upload API

↓

Virus Scan

↓

Copyright Detection

↓

Encoding

↓

Thumbnail Generation

↓

Metadata

↓

Storage

↓

CDN
```

Many expensive processing steps happen before anyone can watch the video.

---

## Netflix

Much simpler.

```text
Studio

↓

Encoding

↓

Storage

↓

CDN
```

No public uploads.

No copyright detection for user content.

No thumbnail generation for creators.

---

# 3. Video Processing

Both platforms encode videos into multiple resolutions.

Example:

```text
Original Movie

↓

240p

480p

720p

1080p

4K
```

Both use adaptive bitrate streaming.

So the technology here is almost identical.

---

# 4. What happens when you press Play?

## YouTube

```text
Open Video

↓

Metadata Service

↓

Choose CDN

↓

Download Chunk 1

↓

Play Starts

↓

Download Remaining Chunks
```

---

## Netflix

```text
Open Movie

↓

Playback Service

↓

Nearest CDN

↓

Choose Best Quality

↓

Download Chunk 1

↓

Play Starts

↓

Download Remaining Chunks
```

Looks almost the same.

So why is Netflix considered different?

Because of **scale and optimization**.

---

# 5. Traffic Pattern

This is the biggest difference.

## YouTube

Imagine these videos:

```text
Funny Cat

100 views

----------------

Gaming Video

2,000 views

----------------

Lecture

20 views

----------------

Music

10 Million views
```

Every video has different popularity.

Most videos are rarely watched.

This is called the **long tail**.

YouTube has billions of videos with highly unpredictable demand.

---

## Netflix

Suppose a new season of *Stranger Things* is released.

```text
Millions of users

↓

Same Episode

↓

Same Time
```

Many users watch the same content simultaneously.

Netflix can aggressively cache those popular videos close to users.

---

# 6. CDN Usage

Both use CDNs.

## YouTube

```text
Google Data Center

↓

Google CDN

↓

User
```

Popular videos are cached.

Less popular videos may come from farther away.

---

## Netflix

```text
Netflix Data Center

↓

Open Connect

↓

ISP

↓

User
```

Netflix places **Open Connect** appliances inside or directly connected to ISPs.

Benefits:

* Lower latency
* Less congestion
* Lower bandwidth costs
* Less buffering

---

# 7. Adaptive Streaming

Both platforms do this.

Suppose you start watching at:

```text
100 Mbps
```

You receive:

```text
4K
```

Internet drops to:

```text
5 Mbps
```

Both YouTube and Netflix switch to:

```text
1080p

↓

720p

↓

480p
```

without stopping playback.

---

# 8. Video Chunks

Neither platform sends the entire movie.

Instead:

```text
Movie

↓

Chunk 1

Chunk 2

Chunk 3

Chunk 4
```

Each chunk is typically a few seconds long.

Advantages:

* Fast startup
* Easier retries
* Adaptive quality changes
* Resume playback

---

# 9. Recommendation

## YouTube

Recommendations depend on:

* Likes
* Comments
* Shares
* Subscriptions
* Watch history
* Trending
* Fresh uploads

Example:

```text
Watched:

Java Tutorial

Spring Boot

Docker

↓

Recommended:

Kubernetes

System Design
```

---

## Netflix

Recommendations focus on viewing behavior.

Example:

```text
Watched:

Money Heist

Narcos

Breaking Bad

↓

Recommended:

Ozark
```

Netflix also considers:

* Completion percentage
* Rewatches
* Genre preference
* Time of day
* Device

---

# 10. Social Features

## YouTube

Includes:

```text
Like

Comment

Subscribe

Share

Playlist

Creator Channel
```

These generate huge amounts of additional traffic.

---

## Netflix

Mostly focused on watching.

```text
Play

Pause

Continue Watching

My List

Ratings (in some regions)
```

No creator ecosystem.

---

# 11. Main Engineering Focus

## YouTube Engineers

Spend significant effort on:

* Upload pipeline
* Copyright detection
* Thumbnail generation
* Live streaming
* Search
* Comments
* Recommendations
* Creator analytics

---

## Netflix Engineers

Spend significant effort on:

* Global CDN
* Open Connect
* Playback quality
* Adaptive bitrate
* Resume playback
* Watch history synchronization
* Recommendation engine
* Subscription and billing

---

# Complete Flow Comparison

### YouTube

```text
Creator Uploads Video
        │
        ▼
Upload Service
        │
        ▼
Virus Scan
        │
        ▼
Copyright Check
        │
        ▼
Video Encoding
        │
        ▼
Thumbnail Generation
        │
        ▼
Storage
        │
        ▼
CDN
        │
        ▼
Viewer Streams Video
```

---

### Netflix

```text
Studio Uploads Movie
        │
        ▼
Encoding Pipeline
        │
        ▼
Storage
        │
        ▼
Open Connect CDN
        │
        ▼
Playback Service
        │
        ▼
Adaptive Bitrate
        │
        ▼
Viewer Streams Movie
```

---

# Interview Summary (Easy to Remember)

| Feature                       | YouTube                                              | Netflix                                                            |
| ----------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------ |
| Public uploads                | ✅ Yes                                                | ❌ No                                                               |
| Encoding                      | ✅ Yes                                                | ✅ Yes                                                              |
| Chunk-based streaming         | ✅ Yes                                                | ✅ Yes                                                              |
| Adaptive bitrate              | ✅ Yes                                                | ✅ Yes                                                              |
| CDN                           | ✅ Yes                                                | ✅ Yes (Open Connect)                                               |
| Open Connect inside ISPs      | ❌ No                                                 | ✅ Yes                                                              |
| Comments & creators           | ✅ Yes                                                | ❌ No                                                               |
| Subscription billing          | Optional                                             | Core feature                                                       |
| Recommendation                | Content discovery                                    | Viewer retention                                                   |
| Primary engineering challenge | Handling billions of uploads and social interactions | Delivering high-quality streams to millions with minimal buffering |

## The easiest way to remember

* **YouTube = "Content Factory + Streaming."** Its unique challenge is accepting and processing an enormous number of user uploads while supporting creators and social features.

* **Netflix = "Streaming Factory."** It has far fewer uploads, so it invests heavily in delivering a smaller catalog with exceptional playback quality using technologies like Open Connect, adaptive bitrate streaming, and highly optimized global caching.

This is why, in system design interviews, **YouTube questions emphasize the upload pipeline**, whereas **Netflix questions emphasize the playback and streaming pipeline**.

Excellent question. This is exactly the kind of question interviewers like because it shows you understand the **business model**, not just the technology.

The short answer is:

> **Netflix doesn't need automatic copyright detection because it already owns or has licensed every video before it is uploaded.**

Let's understand this from the beginning.

---

# YouTube Scenario

Imagine you have a YouTube account.

You upload:

* An Avengers movie
* An Arijit Singh song
* A Premier League match

```text
You

↓

Upload Video

↓

YouTube
```

Can YouTube simply publish it?

**No.**

It doesn't know whether:

* You created the video.
* You copied it from somewhere else.
* You own the rights.

So YouTube must verify this automatically.

---

## Why?

Because anyone can upload anything.

For example:

```text
1. My birthday video ✅

2. My coding tutorial ✅

3. Entire Avengers movie ❌

4. IPL Final recording ❌

5. Taylor Swift song ❌
```

YouTube cannot manually review millions of uploads every day.

So it uses automated copyright detection (such as Content ID) to compare uploaded content against copyrighted material.

---

# Netflix Scenario

Now imagine Netflix.

Who uploads videos?

Not you.

Not me.

Only:

* Netflix Studios
* Disney (if licensed)
* Sony
* Warner Bros.
* Other licensed production companies

```text
Movie Studio

↓

Netflix Content Team

↓

Netflix Platform
```

Netflix already has a legal agreement before the movie is uploaded.

So Netflix already knows:

> "We are allowed to stream this movie."

There is no need to detect copyright because the rights have already been verified through contracts.

---

# Real-Life Analogy

Imagine a bookstore.

### YouTube is like a public notice board.

Anyone can pin anything.

Someone might pin:

* Their own poem ✅
* Someone else's copyrighted novel ❌

The owner needs rules to remove unauthorized content.

---

### Netflix is like an official bookstore.

Every book on the shelf has already been purchased from the publisher.

The bookstore doesn't need to check whether it owns each book every time a customer buys one.

The legal verification happened before the books arrived.

---

# Why doesn't Netflix scan every movie anyway?

Suppose Netflix licenses **Stranger Things**.

Before it appears on Netflix:

* Contracts are signed.
* Rights are purchased.
* Regions where it can be streamed are agreed upon.
* Duration of the license is defined.

Example:

```text
Studio

↓

License Agreement

↓

Netflix

↓

Movie Uploaded
```

Since Netflix controls what enters its catalog, automated copyright detection isn't necessary.

---

# Then what does Netflix verify?

Instead of copyright, Netflix focuses on:

### Video quality

* Is the file corrupted?
* Is the audio synchronized?
* Is the 4K version correct?
* Are subtitles present?

---

### Encoding

Convert the master file into:

```text
4K

1080p

720p

480p

240p
```

---

### DRM (Digital Rights Management)

Prevent users from downloading or copying movies illegally.

Example:

```text
Encrypted Movie

↓

Authorized Netflix App

↓

Decryption

↓

Playback
```

DRM is about **protecting licensed content after distribution**, not checking who owns it.

---

# What if Netflix accidentally uploads a copyrighted movie?

It generally won't happen because content enters Netflix through a controlled business process.

The workflow is typically:

```text
Business Team

↓

Legal Team

↓

License Signed

↓

Content Team

↓

Encoding

↓

Netflix
```

The legal ownership is established before technical processing begins.

---

# Does Netflix ever remove movies?

Yes.

Sometimes licenses expire.

Example:

```text
Netflix licenses Movie X

↓

License valid for 3 years

↓

3 years pass

↓

Movie removed
```

This isn't a copyright detection issue—it's a licensing management issue.

---

# Interview Comparison

## YouTube

```text
Anyone uploads

↓

Need Copyright Detection

↓

Allowed?

↓

Yes → Publish

No → Block/Monetize/Remove
```

---

## Netflix

```text
Licensed Studio

↓

Contract Signed

↓

Upload

↓

Encode

↓

Publish
```

No copyright detection step is needed because only authorized content enters the platform.

---

# Beginner Summary

| YouTube                                     | Netflix                                                    |
| ------------------------------------------- | ---------------------------------------------------------- |
| Anyone can upload videos                    | Only Netflix or licensed studios upload                    |
| Must detect stolen content automatically    | Rights are verified through legal agreements before upload |
| Uses copyright detection systems            | Uses licensing management instead                          |
| Protects creators from unauthorized uploads | Protects licensed content using DRM and access controls    |

## Easy way to remember

* **YouTube asks:** *"Does this uploader own this video?"*
* **Netflix asks:** *"Do we still have a valid license to stream this video in this region?"*

That's the key difference. **YouTube's challenge is copyright verification at upload time, while Netflix's challenge is license management and secure content delivery after the content has already been legally acquired.**
