---
title: "Web Crawling In Golang With Colly"
subTitle: "Learn Golang By Doing"
date: 2020-09-25T00:10:00+03:00
draft: false

# post thumb
image: "images/Golang/colly-web-crawler.png"

# meta description
author: "Gal Zaidman"
description: "We will learn what is web crawling and how to implement a simple web crawler in golang using the colly framework"

# taxonomies
categories:
  - "Golang"

tags:
  - "Golang"
  - "WebCrawling"
  - "Colly"

# post type
type: "post"
---

### Before we start

This tutorial was created while implementing a small project that required web crawling.
It is composed from various descriptions, definitions and examples I saw on the links in the References section as well as my own experience while working on the project.

### What is a web crawler?

Essentially, a web crawler is a tool that inspects the HTML of a givin web page and performs some type of actions based on that content. On the simple case (which is what we will implement in this tutorial) the web crawler start from a simple page, and while scanning that page it acquire more links to visit.
Lets look at the following pseudo code to better understand the basics:

```python
Queue = Queue()
Queue.Add(initialURL)

while Queue is not empty:
    URL = Queue.pop()
    Page = Visit(URL)
    Links = ExtractURLs(Page)
    for link in Links:
        Queue.Add(link)
```

Usually when we think of writing wab crawlers the first language tha comes to mind is python, with its wide selection of frameworks and rich data processing abilities, however Golang has a very good rich ecosystem for web crawling as well that lets you utilize the efficiency of Golang.

### Colly - The golang web crawling framework

#### Introduction

Colly is a Golang framework for building web scrapers. With Colly you can build web scrapers of various complexity, from simple scraper to complex asynchronous website crawlers processing millions of web pages. Colly is very much "Batteries-Included", meaning you will get the most required features "Out of the box".
Colly has a rich API with features such as:

- Manages request delays and maximum concurrency per domain
- Automatic cookie and session handling
- Sync/async/parallel scraping
- Distributed scraping
- Caching
- Automatic encoding of non-unicode responses

To install Colly we need to have Golang installed and run:

```bash
go get -u github.com/gocolly/colly/...
```

Then in our go file we need to import it:

```go
import "github.com/gocolly/colly"
```

Latest info can be found in [colly installation guide](http://go-colly.org/docs/introduction/install/)

#### Basic Components

##### Collector

Colly’s main entity is the [Collector struct](https://pkg.go.dev/github.com/gocolly/colly/v2?tab=doc#Collector). The Collector keep track of pages that are queued to visit, manages the network communication and responsible for the execution of the attached callbacks when a page is being scraped.

To initialize a Collector we need to call the NewCollector function:

```go
// Create a collector with the default configuration
c := colly.NewCollector()
```

The NewCollector function definition is

```go
func NewCollector(options ...CollectorOption) *Collector
```

CollectorOption is a function which accepts a pointer to a Collector and configures it

```go
type CollectorOption func(*Collector)
```

We can basically configure each field in the Collector struct, for example here is a collector that is Asynchronous and will only go to links on the starting page:

```go
c := colly.NewCollector(
    // MaxDepth is 2, so only the links on the scraped page
    // and links on those pages are visited
    colly.MaxDepth(2),
    colly.Async(true),
)
```

Important Notes:

- We can override our configuration at any point.
- We can use environment variables to configure our collector, this is useful if we don't want to recompile the program for customizing the collector, but we need to remember that every configuration that is defined in our program will override our environment variable.

You can see the full list of CollectorOption functions in [colly goDocs](https://pkg.go.dev/github.com/gocolly/colly/v2?tab=doc#CollectorOption) and for more detailed information about how we can configure the collector go the the [colly configuration docs](http://go-colly.org/docs/introduction/configuration/)

##### Callbacks

Colly gives us a number of callbacks that we can setup. Those callbacks will be called on various stages in our crawling job and you will need to think which one do you want based on you requirements.
Here is a list of all the callbacks and the order in which they will be called (taken from the colly website[1]):

1. OnRequest(f RequestCallback) - Called before a request.
2. OnError(f ErrorCallback) - Called if error occured during the request.
3. OnResponse(f ResponseCallback) - Called after response received.
4. OnHTML(goquerySelector string, f HTMLCallback) - Called right after OnResponse if the received content is HTML.
5. OnXML(xpathQuery string, f XMLCallback) - Called right after OnHTML if the received content is HTML or XML.
6. OnScraped(f ScrapedCallback) - Called after OnXML callbacks.

Each of those callbacks receive a function which will be triggered when the callback is called.

Now that we understand the basics of colly, lets dive into our project

#### Our project

##### Overview

So under the openshift organization we have various github repos. Merged PRs will trigger a release build. Whenever QE needs to verify a Bug which is resolved in a certain PR they need to manually look at when the PR was merged and then go to Openshift release page and find a release job which was triggered after the time the PR was merged, check its page and see that it contains a the PR and then they know that they can use the release for verification. We want to automate this task.

We will build a small project that checks when a PR was merged, and then crawl on the openshift release page and find the release which contain the PR.

##### Logic

1. Retrieve the PR from the user.
2. Validate that it is a github PR link.
3. Find when the PR was merged.
4. Find all the links to release pages on the openshift release.
5. On each link search for the PR ID.

##### Implementation

So we will use:

- flags - for parsing our commend line args.
- regexp - to validate the pattern of a github repo.
- github api - to retrieve the PR data.

Our main function will look like:

```go
func main() {
	var err error
	debug := flag.Bool("debug", false, "run with debug")
	flag.Parse()
	url := flag.Arg(0)
	prd, err = newPullRequestData(url)
	if err != nil {
		panic(err)
	}
	setupReleasePageCrawler(*debug)
	setupReleaseStatusCrawler(*debug)
	releaseStatusCrawler.Visit("https://" + OPENSHIFT_RELEASE_DOMAIN)
	releasePageCrawler.Wait()
	fmt.Println("Done!")
}
```

We get the URL from the user and also configure a flag for running in DEBUG mode.
Then we get the PR details and configure our crawlers

- releaseStatusCrawler will be in charge of parsing the release domain and finding relevant release pages.
- releasePageCrawler will be in charge of finding the PR in the release page.

And at the end we visit the release page, and call the Wait function on releasePageCrawler since it will run asynchronously.

First of all, lets create a struct that will hold all of the data which we need for a PR.

```go
// PullRequestData holds all the data needed for a PR
type PullRequestData struct {
	org   string
	repo  string
	id    int
	mDate time.Time
}
```

Then write a the logic for getting a PR Url and creating a PullRequestData object.

```go
// Gets a PR URL and returns the organization, repo and ID of that PR.
// If the URL is not a valid PR url then returns an error
func parsePrURL(prURL string) (string, string, int, error) {
	githubReg := regexp.MustCompile(`(https*:\/\/)?github\.com\/(.+\/){2}pull\/\d+`)
	if !githubReg.MatchString(prURL) {
		return "", "", 0, fmt.Errorf("ERROR: prURL is not a github PR URL, got url %v", prURL)
	}
	u, err := url.Parse(prURL)
	if err != nil {
		return "", "", 0, err
	}
	pArr := strings.Split(u.Path, "/")
	if len(pArr) != 5 {
		return "", "", 0, fmt.Errorf("Expected PR URL with the form of github.com/ORG/REPO/pull/ID, but instead got %v", prURL)
	}
	id, err := strconv.Atoi(pArr[4])
	if err != nil {
		return "", "", 0, fmt.Errorf("ERROR: Expected PR URL with the form of github.com/ORG/REPO/pull/ID, but instead got %v", prURL)
	}
	return pArr[1], pArr[2], id, nil
}

// Creates a Github client using AccessToken if it exists or an un authenticated client
// if no AccessToken is available and retrieves the PR details from github
func getGithubPrData(org string, repo string, id int) (*github.PullRequest, error) {
	var client *github.Client
	ctx := context.Background()
	accessToken := os.Getenv("AccessToken")
	if accessToken == "" {
		client = github.NewClient(nil)
	} else {
		ts := oauth2.StaticTokenSource(
			&oauth2.Token{AccessToken: accessToken},
		)
		tc := oauth2.NewClient(ctx, ts)
		client = github.NewClient(tc)
	}
	pr, _, err := client.PullRequests.Get(ctx, org, repo, id)
	return pr, err
}

func newPullRequestData(prURL string) (*PullRequestData, error) {
	org, repo, id, err := parsePrURL(prURL)
	if err != nil {
		return &PullRequestData{}, err
	}
	pr, err := getGithubPrData(org, repo, id)
	if err != nil {
		return &PullRequestData{}, err
	}
	return &PullRequestData{
		org:   org,
		repo:  repo,
		id:    id,
		mDate: pr.GetMergedAt(),
	}, nil
}
```

Here we get a PR URL, then call parsePrURL which validates that the URL is correct and returns the  organization, repository and id from the URL. After we retrieved the fields from the PR URL we call getGithubPr which creates a github client and uses github API for getting the PR details. Finally we return a pointer to a PullRequestData object.

Now lets configure our first crawler, the release status crawler.
The release status crawler works on one page only, the OCP release page which contains ~2000+ links.

```go
func setupReleaseStatusCrawler(debug bool) {
	releaseStatusCrawler = colly.NewCollector(
		colly.AllowedDomains(OPENSHIFT_RELEASE_DOMAIN),
		colly.MaxDepth(1),
	)
	if debug == true {
		releaseStatusCrawler.OnRequest(func(r *colly.Request) {
			fmt.Println("Debug: release status crawler is visiting: ", r.URL.String())
		})
	}
	releaseStatusCrawler.OnHTML("a[href]", filterReleasesLinks)
}
```

Here we create a new crawler that can visit only ocp domain and set MaxDepth to 1 so it will stay only one first link.
Then we will use the OnRequest callback to print the URL we are visiting in debug mod.
Finally we use the OnHTML callback to set a function that will be called on each a\[href\] object.
filterReleasesLinks is a function of type [HTMLCallback](https://pkg.go.dev/github.com/gocolly/colly?tab=doc#HTMLCallback).
It gets the a\[href\] object and triggers the releasePageCrawler (which works on async mode) if that element fits the release regex and is created after the PR is merged.

```go
func filterReleasesLinks(e *colly.HTMLElement) {
	reRelease := regexp.MustCompile(`\d\.\d\.\d-\d\.(nightly|ci)-\d{4}-\d{2}-\d{2}-\d{6}`)
	if !reRelease.MatchString(e.Text) {
		return
	}
	d := strings.SplitN(e.Text, "-", 3)[2]
	d = strings.Split(d, " ")[0]
	tr, err := time.Parse(PR_DATE_FORMAT, d)
	if err != nil {
		fmt.Printf("Error time.Parse: ", err)
	}
	// Check if PR merge time t is after the release creation time tr
	if prd.mDate.After(tr) {
		return
	}
	link := e.Attr("href")
	releasePageCrawler.Visit(e.Request.AbsoluteURL(link))
}
```

Now lets configure our second crawler, the release page crawler.
The release page crawler works on one page only, the release page which should contain a link to the given PR.
The release page crawler will be triggered by the release status crawler when it finds a relevant link, so it needs to work on Async mode.

```go
func setupReleasePageCrawler(debug bool) {
	releasePageCrawler = colly.NewCollector(
		colly.AllowedDomains(OPENSHIFT_RELEASE_DOMAIN),
		colly.MaxDepth(1),
		colly.Async(true),
	)
	//Set max Parallelism and introduce a Random Delay
	releasePageCrawler.Limit(&colly.LimitRule{
		Parallelism: 6,
		RandomDelay: 1 * time.Second,
	})
	if debug == true {
		releasePageCrawler.OnRequest(func(r *colly.Request) {
			fmt.Println("Debug: release page crawler is visiting: ", r.URL.String())
		})
	}
	releasePageCrawler.OnHTML("a[href]", findPR)
}
```

Here we create a new crawler that can visit only ocp domain ,set MaxDepth to 1 so it will stay only on the first link and configure it to work in Async mode.
Then we set a limit to be a good citizen of the web, this is important because websites can block users that overload the site.
Then we will use the OnRequest callback to print the URL we are visiting in debug mod.
Finally we use the OnHTML callback to set a function that will be called on each a\[href\] object.
findPR is also a [HTMLCallback](https://pkg.go.dev/github.com/gocolly/colly?tab=doc#HTMLCallback) function.
It gets the a\[href\] object and if it is a valid PR URL checks if it has the same ID as the given PR.
If the IDs match we know that we found a release which contains our PR and we print it for the user.

```go
func findPR(e *colly.HTMLElement) {
	link := e.Attr("href")
	_, _, id, err := parsePrURL(link)
	if err != nil {
		return
	}
	if prd.id == id {
		fmt.Println("found release, link ", e.Request.URL.String())
	}
}
```

The full code can be found on:
https://github.com/Gal-Zaidman/ocp-release-finder

### References:

[1] [Colly web page and docs](http://go-colly.org/)
[2] [Scraping the Web in Golang with Colly and Goquery](https://benjamincongdon.me/blog/2018/03/01/Scraping-the-Web-in-Golang-with-Colly-and-Goquery/)