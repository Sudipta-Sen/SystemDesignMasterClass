# Live Streaming Overview

## Example: Cricket Streaming

To understand live streaming, we will use the example of cricket streaming. Contrary to common belief, there is a significant delay of around 10-15 seconds between what happens in the stadium and what we see on TV or the internet. This delay occurs due to the process of capturing, processing, and transmitting video feeds.

- **The Process:**
    1. Capturing the Feed:
        - Multiple cameras capture the match in the stadium.
        - In the control room, a director views feeds from all cameras and selects the best angles, switching as necessary (e.g., switching to a camera showing the right side after the batsman hits the ball there).
        - The chosen footage forms the final stadium feed.
    
    2. Broadcasting the Stadium Feed:

        - The stadium feed is sent to various broadcasters: TV channels, radio stations, OTT platforms (like Hotstar).
        - OTT platforms receive the stadium feed and distribute it to users.

    3. Handling Feeds at OTT Servers:

        - The stadium feed does not contain advertisements. Ads are injected later by broadcasters or OTT platforms (not by the stadium itself).
        - The OTT platform, such as Hotstar, streams the stadium feed to the users' devices without ads in some cases.
    
    4. Chunking the Stream:

    - The stadium feed is divided into 10-second chunks.
    - The user selects a match on the OTT platform (e.g., Hotstar), and the video player on their device requests the video chunks.
    - To minimize delay, OTT platforms use Content Delivery Networks (CDNs) to store video chunks closer to the user, ensuring faster access and smoother streaming.

    5. How Video Streaming Works:

        - Pull vs. Push: The video player pulls data (chunks) from the CDN, rather than the CDN pushing data. This method is more reliable in case of connection issues or missed chunks.
        - Video players buffer the data, displaying the next chunk as it's received.
        - If a chunk is delayed or unavailable, the video player shows a buffering icon, indicating the wait for the next chunk.

    6. The Role of the M3U8 File:

        - The M3U8 file is a metadata file containing the sequence and names of video segments (e.g., seg000001.seg, seg000006.seg).
        - When the video player starts, it first downloads the M3U8 file to understand the video’s structure (e.g., chunk size, resolutions).
        - For live streaming, the M3U8 file updates every 5 seconds, helping the CDN serve the next chunk.

    7. CDN and S3 Integration:

        - The CDN fetches the video chunks from the origin server (often hosted on S3).
        - Hotstar servers process high-resolution feeds and store them in S3.
        - Multiple resolutions are created (e.g., 360p, 720p, 1080p) to adapt to user devices and bandwidth.
    
    8. Handling Failures:

        - If Hotstar servers go down, the S3 server and Lambda functions handle video processing and chunk storage.
        - CDNs poll for updates from S3 and deliver data to users.

## Securing Video Streaming:
1. Authentication for Subscribed Users:

    - To restrict access to subscribed users, a certification server issues short-lived certificates to users.
    - The user's device must provide the certificate to the CDN to access the video chunks.
    - The certificate is renewed periodically, ensuring that unauthorized users cannot access the content.

2. High Availability of the Certification Server:

    - The certification server is crucial for user authentication and must be highly available.
    - It verifies subscription status by querying the user database, and a cache is used to reduce the load on the database.

## Handling Advertisements in Live Streaming:
1. Types of Ads:

    - Video Ads: Shown between overs (e.g., 15-second video ads).
    - Popup Ads: Appear temporarily during the match.
    - Server-Rendered Ads: Inserted into the match feed, replacing the live feed during breaks or pauses.

2. Differences in Ad Placement:

    - In on-demand videos (e.g., YouTube), the video pauses for ads, and the content resumes afterward.
    - In live streaming, the match continues while ads are shown, meaning viewers miss a portion of the live event during the ad.

3. Server-Rendered Ads:

    - Ads are inserted into the live stream after the stadium feed is processed by the OTT platform (e.g., Hotstar).
    - These ads become part of the final feed, even when users rewind the stream later.

## Redundancy in CDNs:
- OTT services use multiple CDNs to distribute traffic.
- If one CDN goes down, the traffic is routed to another, ensuring uninterrupted service.

## Video Processing and Resolution Scaling:
- High-performance systems are required to process multiple resolutions of the video (e.g., 360p, 720p, 1080p) in real-time, allowing for smooth live streaming.
- Hotstar processes the stadium feed and pushes the chunks to S3.

In summary, live streaming involves multiple components—capturing stadium feeds, processing chunks, storing them in CDNs, and handling user authentication—while also dealing with complexities like adding ads and scaling for millions of viewers.