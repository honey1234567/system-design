The YouTube system design problem is one of the **most frequently asked Senior SDE system design interviews** because it covers almost every distributed system concept:

https://systemdesignschool.io/problems/youtube/solution?utm_source=neetcode

* Object Storage
* CDN
* Queues
* Async Processing
* Caching
* Blob Storage
* Scaling
* Distributed Workers
* Parallel Processing
* Cost Optimization
* High Availability

The article focuses on **Video-on-Demand (VOD)** like YouTube (not live streaming). ([systemdesignschool.io][1])

---

# 1. First Understand the Problem

Suppose you upload a **2GB movie**.

Naively you might think:

```
User
   |
Upload Video
   |
App Server
   |
Store File
   |
Other users download
```

Looks simple.

But imagine

* 100 Million users
* 500 hours uploaded every minute
* Billions of views every day

This simple architecture immediately fails.

The article explains how we gradually improve it.

---

# 2. Functional Requirements

Very few requirements.

### Upload video

User uploads

```
movie.mp4
```

---

### Watch video

Users watch

```
https://youtube.com/watch?v=123
```

---

### Smooth playback

Video should not stop every few seconds.

---

### Seek

Jump to

```
5 min

10 min

1 hour
```

without downloading whole video.

---

# 3. Biggest Difference Between YouTube and Instagram

Instagram

```
Image

Few MB
```

YouTube

```
Video

5GB
10GB
50GB
```

Huge files.

Storage is not the biggest issue.

**Network transfer (egress)** is.

The article emphasizes that **serving video bytes to viewers dominates cost**, much more than storing them. ([systemdesignschool.io][1])

---

# 4. Why can't we store one video file?

Suppose you upload

```
Vacation.mp4

4K
```

Should everyone receive 4K?

No.

Imagine

Person A

```
Fiber Internet
```

Needs

```
4K
```

Person B

```
Slow 3G
```

Cannot play 4K.

Person C

```
Mobile
```

Needs

```
480p
```

Therefore one uploaded video becomes many versions.

---

# 5. Encoding Ladder (Very Important)

This is probably the most important concept.

Suppose original upload

```
4K
```

Server creates

```
240p

480p

720p

1080p

4K
```

These are called

> **Renditions**

The complete set is called

> **Encoding Ladder**

Example

```
Original

      |
      V

+-------------+
| 4K Upload   |
+-------------+

      |

--------------------------------

240p

480p

720p

1080p

4K
```

Why?

Different devices.

Different internet speeds.

Different screen sizes.

([systemdesignschool.io][1])

---

# 6. What is Transcoding?

Suppose user uploads

```
holiday.mov
```

TV may not support it.

Android may not support it.

Browser may not support it.

So YouTube converts it.

This conversion is

```
Upload

↓

Decode

↓

Compress

↓

Resize

↓

Encode

↓

Multiple formats
```

called

> **Transcoding**

Example

```
Input

4K

↓

Output

240p

480p

720p

1080p
```

---

# 7. Why not transcode immediately?

Imagine

```
Upload

↓

Server starts converting

↓

30 minutes
```

User waits.

Impossible.

Instead

```
Upload

↓

Save

↓

Queue

↓

Worker

↓

Transcode

↓

Ready
```

Much faster.

This is asynchronous processing. ([systemdesignschool.io][1])

---

# 8. Why Queue?

Suppose

1000 users upload together.

Without queue

```
App Server

↓

Transcoding

↓

CPU 100%
```

Server crashes.

Instead

```
Uploads

↓

Queue

↓

Workers

↓

Process slowly
```

Workers pick jobs one by one.

Example queue

```
Video1

Video2

Video3

Video4

Video5
```

Workers

```
Worker1

Worker2

Worker3
```

Each takes one job.

---

# 9. Blob Storage

Videos are huge.

Never store inside database.

Instead

```
Database

Video metadata

----------------

id

title

description

duration
```

Actual video

```
Object Storage

movie.mp4
```

Examples

* Amazon S3
* Azure Blob Storage
* Google Cloud Storage

---

# 10. Why Direct Upload?

Wrong approach

```
User

↓

App Server

↓

S3
```

Problem

4GB travels through server.

Waste.

Correct

```
App Server

↓

Generate Signed URL

↓

User uploads directly

↓

Object Storage
```

Application server never handles the 4GB.

It only handles metadata and permissions. ([systemdesignschool.io][1])

---

# 11. Chunk Upload

Suppose uploading

```
5GB
```

Internet disconnects at

```
4.9GB
```

Without chunking

```
Restart

0%
```

Terrible.

Instead

Split into

```
Chunk1

Chunk2

Chunk3

Chunk4

Chunk5
```

If chunk4 fails

Only resend

```
Chunk4
```

This is resumable upload. ([systemdesignschool.io][1])

---

# 12. Why Split Video into Segments?

Video streaming does **not** download the whole movie first.

Instead

```
Movie

↓

4 sec

↓

4 sec

↓

4 sec

↓

4 sec
```

Small pieces.

Example

```
segment1.ts

segment2.ts

segment3.ts

segment4.ts
```

Player downloads continuously.

---

# 13. Manifest File

How does player know

* available qualities?
* segment names?

Using

Manifest

Example

```
Video

240p

segment1

segment2

segment3

-----------------

720p

segment1

segment2

segment3

-----------------

1080p

segment1

segment2

segment3
```

Formats include HLS (`.m3u8`) and MPEG-DASH (`.mpd`). The player first downloads the manifest, then requests segments. ([systemdesignschool.io][1])

---

# 14. Buffer

Suppose network becomes slow.

Without buffer

```
Play

Stop

Play

Stop
```

Buffer stores future video.

```
Downloading

██████████

Playing

██
```

If internet slows

Player continues using

```
Buffer
```

No interruption.

---

# 15. Adaptive Bitrate (ABR)

One of the most important concepts.

Suppose network speed

```
10 Mbps
```

Play

```
1080p
```

Now bandwidth becomes

```
2 Mbps
```

Player switches

```
1080p

↓

720p

↓

480p
```

without stopping.

When speed improves

```
480p

↓

720p

↓

1080p
```

This decision is made **by the client/player**, based on the manifest and current bandwidth—not by the server. ([systemdesignschool.io][1])

---

# 16. CDN

Without CDN

```
India

↓

USA Server
```

Every request

Very slow.

Instead

```
USA Origin

↓

India CDN

↓

Users
```

First request

```
Origin

↓

CDN
```

Next million users

```
CDN only
```

Origin is untouched.

Huge savings.

---

# 17. Metadata vs Media

Very important interview point.

Database

```
Video ID

Uploader

Views

Likes

Title

Duration
```

Object Storage

```
Actual Video

Thumbnail

Segments
```

Never store binary video in SQL.

---

# 18. Parallel Transcoding

Suppose

```
2 hour movie
```

Don't let one machine encode it from start to finish.

Instead

```
Movie

↓

Split

↓

Segment1 → Worker1

Segment2 → Worker2

Segment3 → Worker3

Segment4 → Worker4
```

Parallel work dramatically reduces total processing time. Failed segments can be retried independently instead of restarting the whole video. ([systemdesignschool.io][1])

---

# 19. High-Level Architecture

```
                Upload

                  |

             Load Balancer

                  |

            Upload Service

                  |

          Generate Signed URL

                  |

          Object Storage (Raw Video)

                  |

          Upload Completed Event

                  |

             Message Queue

                  |

        -----------------------
        |         |           |
     Worker1   Worker2    Worker3

        |         |           |

     Transcoding + Segmentation

                  |

         Object Storage (Renditions)

                  |

          Generate Manifest

                  |

               CDN

                  |

              End Users
```

---

# 20. Why This Design Scales

* **Application servers** don't handle multi-GB uploads.
* **Object storage** stores massive files reliably.
* **Queues** absorb upload spikes.
* **Worker fleets** transcode in parallel.
* **Multiple renditions** support different devices and network speeds.
* **Segmented videos** allow seeking and adaptive streaming.
* **CDNs** serve most requests, minimizing expensive origin bandwidth.
* **Databases** store lightweight metadata, while media stays in object storage. ([systemdesignschool.io][1])

---

# Common Interview Questions

| Question                              | Expected Answer                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Why use object storage instead of DB? | Videos are huge binary objects; databases are optimized for structured metadata.                 |
| Why use a queue?                      | Transcoding is CPU-intensive and should be asynchronous.                                         |
| Why split uploads into chunks?        | Resume interrupted uploads without restarting.                                                   |
| Why split videos into segments?       | Efficient streaming, seeking, and CDN caching.                                                   |
| Why create multiple renditions?       | Different devices and bandwidths require different qualities.                                    |
| Why use ABR?                          | Automatically adjust video quality to changing network conditions.                               |
| Why use a CDN?                        | Reduce latency, lower origin load, and cut bandwidth costs.                                      |
| Why keep the original uploaded video? | Future re-transcoding with better codecs or resolutions without asking the user to upload again. |

This design pattern—**direct upload → object storage → async processing via queue → multiple renditions → segmented media → CDN delivery with client-side adaptive bitrate**—is the core architecture used by modern video-on-demand platforms such as YouTube and similar services. ([systemdesignschool.io][1])

[1]: https://systemdesignschool.io/problems/youtube/solution?utm_source=chatgpt.com "YouTube System Design | System Design Interview"
