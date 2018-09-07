---
layout: post
title: "Scraping Public Data For Treasure"
date: 2018-08-27 23:59
comments: true
author: Trevor Steen
authorIsRacker: true
published: true
categories:
  - Security
  - Automation
  - Python
---

 [Pastebin](https://pastebin.com/) and [GitHub](https://github.com) (and their Gist feature) have long been mechanisms for sharing data for both benign and nefarious purposes. System admins may use [Pastebin](https://pastebin.com/) to share a configuration file with a customer or Gist to share a code snippet to fix a bug. Attackers can use these same tools to anonymously share data with other untrusted parties or to publish leaks and hacks from recent endeavors.

 Organizations must remain vigilant to determine if any of their proprietary or sensitive information is being shared either due to carelessness or malicious intent

 <!-- more -->

# Pastebin

We have developed a set of scripts to do just this.  Our first attempt at scraping [Pastebin](https://pastebin.com/) for data used [this](https://github.com/fabiospampinato/pastebin-monitor) simple script to look for interesting data.  Unfortunately at the time, we were only able to deploy the tool as a Slack integration with default rules and ended up with a lot of noise that was not immediately relevant to us.  Additionally, most [Pastebin](https://pastebin.com/) scrapers that are found on [GitHub](https://github.com) make use of old [Pastebin](https://pastebin.com/) endpoints and do not utilize [Pastebin Pro accounts](https://pastebin.com/pro).

## Scraping API

Looking into the [Pastebin API Docs](https://pastebin.com/api) gives a lot of details about how to create, list, and delete pastes that a user has access to.  Hidden at the bottom of the documentation, though, is an interesting gem: the [Scraping API](https://pastebin.com/doc_scraping_api). The [Scraping API](https://pastebin.com/doc_scraping_api) is designed to be an easier options for programmatic use of the [Pastebin](https://pastebin.com/) platform.  Instead of using the web-based search function and having to parse the raw HTML responses, the [Scraping API](https://pastebin.com/doc_scraping_api) gives us exactly what we need in JSON format for easy parsing.

The only limitations are that only 1 IP can be connected to a [Pro](https://pastebin.com/pro) account at a time and a recommendation to limit requests to 1 per second.

## Implementation

In our implementation of the scraper, 100 of the most recent pastes are parsed each minute in accordance with [Pastebin's Scraping API](https://pastebin.com/doc_scraping_api) recommendation.  Because the response from the scraping API does not include paste contents (just the metadata), we then go and query those new pastes directly by their raw URL. We save the last 300 paste IDs so that each round we are not requesting the same pastes again. The average number of pastes created per minute seems to be around 30-40 with obvious spikes and troughs.

After each paste is retrieved we scan it for keywords and Regular Expressions (REGEXs).  If there are hits for any given run (again once per minute) they have the options to be sent as Splunk events to a Splunk HTTP Event Collector and/or in an email to configured recipients. We send the metadata and the raw text from the paste because in some cases, the data may have been deleted by the time we get around to reviewing the event. If the raw text is longer than 120 lines, we truncate the data and send only the first 10 lines, the 100 lines surrounding the match, and the last 10 lines.  The goal was to run this tool as a stand alone container so long term storage was not a primary concern.

## Findings

To date, our most significant finding has been an SSH configuration file uploaded to [Pastebin](https://pastebin.com/) by an employee that had details about internal domain space and username format.

 As a service provider and technology company we see a lot of benign hits on information like IP address ownership lists and scripts that integrate with our platforms.

 We also found through both documentation and trial/error that pastes do not post immediately.  [Pastebin](https://pastebin.com/) has a 2 minute delay in publishing pastes, presumably to give users time to edit, delete, or take other action.

# Gist

After completing the work on the [Pastebin](https://pastebin.com/) scraper,we had recommendations to also look at Gist. Our Gist scraper is simply a port of the [Pastebin](https://pastebin.com/) scraper code applied to [GitHub Gist](https://gist.github.com).  There are two notable differences: 1) the API calls are updated to work with [GitHub's API](https://developer.github.com/v3/) and 2) The [GitHub rate limit](https://developer.github.com/v3/#rate-limiting) is stricter (enforced 60 requests per hour) than Pastbin's and therefore we pull up to the 100 most recent Gists every 2 minutes instead of 1 minute. I tried using 1 request per minute but at the hour point, if the requests were just on the wrong side of the hour, we would get locked out. Polling at 2 minute intervals solved the problem and did not negatively impact the performance of the tool.

On average we are seeing about 20 new Gists per polling interval.

## Findings

We have not had any major findings from Gist.  Most events are code snippets that integrate with our services and are benign.

# TruffleHog

In the same thread of discovering public data that might be of interest to our security organization, we had played around with [TruffleHog](https://github.com/dxa4481/truffleHog) in the past. The major problem that we had with running [TruffleHog](https://github.com/dxa4481/truffleHog) was that we did not have a way to efficiently run it against multiple [GitHub](https://github.com) repos.  In our case, we have the publicly accessible [Rackerlabs](https://github.com/rackerlabs/) and [Racker](https://github.com/racker) organizations that we wanted to run. With 620 and 219 repos respectively, this was not going to be able to be completed manually.

We decided to write a wrapper for [TruffleHog](https://github.com/dxa4481/truffleHog) that would take a list of organizations and/or repos as input, collect all of the repos included in those organizations and then run [TruffleHog](https://github.com/dxa4481/truffleHog) against them.

## Implementation

As we built this automation, a few things came to light:

1. Because of all the things that [TruffleHog](https://github.com/dxa4481/truffleHog) requests for a given repo to do its analysis, it will burn through your [GitHub rate limit](https://developer.github.com/v3/#rate-limiting) in no time.  The unauthenticated limit of 60 requests per hour simply is not sufficient so all requests need to be authenticated. Even with authenticated requests, with the number of repos we were trying to hit, we burnt through about half of our hourly limit in a single run.
2. TurffleHog is not fast. A single run from scratch took about 4 hours for the 620 repos I was testing against in [Rackerlabs](https://github.com/rackerlabs/).  Adding in some parallelization (multiple repos at once) did not reduce the time significantly as the bulk of the time was spent on the few repos that had monstrous commit histories and [TruffleHog](https://github.com/dxa4481/truffleHog) does not have any internal parallelization. We also added logic to not scan parts of the repos that had already been scanned by writing the current commit hash to a file and using that to pick up where we left off on subsequent scans.
3. [TruffleHog](https://github.com/dxa4481/truffleHog) takes up a TON of disk space.  All of the repos are cloned into `/tmp` and in the main version of the tool are never deleted. I ran out of disk space multiple times.  There is a [pull request](https://github.com/dxa4481/truffleHog/pull/118) to fix this.
4. There are multiple issues with the current version of [TruffleHog](https://github.com/dxa4481/truffleHog), most of which have [pull requests](https://github.com/dxa4481/truffleHog/pulls) pending approval. Because of this, we forked the code and implemented most of the pending pull requests including a major one that corrected an off-by-one error in the reported hash for any findings.

We also found that the standard format for output was not conducive to massive scanning, reporting, and parsing. Because of this, we decided to go with the JSON output that would allow for easier parsing.

With the output in hand, we quickly found that the bulk of data simply was too much to sift through so we ended up writing an additional parser script to parse the JSON output and filter keywords and to only show findings since a specified date.  This allowed us to pare down a few thousand results to about 30.

## Findings

There was a lot of noise in the findings, mostly based on demo, sample, or example code.  All sensitive information that we retrieved appears to be out of date and no longer useful.  We plan to continue using this tool in a repeat audit fashion. Other options would be to add TruffleHog as a stage on a CI/CD pipeline.

# The Scripts

This collection of scripts can be found [here](https://127.0.0.1)
