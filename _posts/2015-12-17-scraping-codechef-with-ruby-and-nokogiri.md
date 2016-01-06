---
layout: post
title:  "Scraping CodeChef with Ruby and Nokogiri"

excerpt: "
A few weeks before Avishkar we held the qualifier round for Insomnia on CodeChef.
The week after that the event coordinators needed to get a list of all the teams who registered for the event and their details.
They needed it urgently so I decided to just scrape what they needed off of CodeChef's website directly.
This post is about how I did that."
tags: [codechef, ruby, nokogiri, web scraping, avishkar, insomnia, mnnit allahabad]
categories: [Scripting]
comments: true
---

So, I haven't made any posts in a long time.
This was party due to laziness and partly because I was busy with other things.
For the past few months it was because I was busy with preparations for [Avishkar](http://avishkar.xyz/)[^1] 2015.

A few weeks before Avishkar we held the qualifier round for Insomnia[^2] on [CodeChef](https://www.codechef.com/INSQ2015).
The week after that the event coordinators needed to get a list of all the teams who registered for the event and their details.
They needed it urgently so I decided to just scrape what they needed off of CodeChef's website directly.
This post is about how I did that.

## Planning

Everything we needed was available on CodeChef's website and we just had to get it from there.
Now obviously collecting all the data of 622 teams manually wasn't a good solution.
We had to automate this task.
The better option was *web scraping*.
From Wikipedia,

> Web scraping (web harvesting or web data extraction) is a computer software technique of extracting information from websites.

Simple enough definition and the process too is simple enough (at least for what I had to do).
I had previously done web scraping with Python and Beautiful Soup so this time I wanted to try it with Ruby.

### Nokogiri

A quick search for a gem that would help me parse HTML yielded Nokogiri.
After taking a look at the tutorial on Nokogiri's website I knew this was exactly what I needed.
Installing it was as simple as,

`gem install nokogiri`

### Finding the data

The list of registered teams is on [this page](https://www.codechef.com/teams/list/INSQ2015).
There are 25 pages but luckily they can be accessed by setting the value of the `page` query parameter.
Unluckily, the table on the pages doesn't have all the information we needed.
We can get the Team Name, Institution and Team members from the table but we also need the Country and Real Name of the members.
For those we'll have to parse the CodeChef profiles of each member.

The next step is to inspect the HTML of both the teams page and profile page.

From the HTML of the teams page we can see that all the data is in a HTML table with class `rank-table`.
Each row of this table has three columns, having the Team Name, Institution and Members.
Also, the Members column has the URL to the profiles of each member too.

From the HTML of the profile page we can see that the Name is in a `div` tag with class `user-name-box` and the Country is in a `span` tag with class `user-country-name`.

## Coding

We now have everything ready to start coding. A copy of the code I wrote (with comments) can be found below.

{% highlight ruby linenos %}
#!/usr/bin/env ruby

require 'nokogiri'
require 'open-uri'

# Download Teams page
# ARGV[0] is the first argument given to the script when it is run
# We could've also just run the whole thing in a loop instead of running the script separately for each page
doc = Nokogiri::HTML(open("https://www.codechef.com/teams/list/INSQ2015?page=" + ARGV[0]))

# Get all the rows of the table with class rank-table
rows = doc.css(".rank-table tr")

# Iterate each row to get details of a team
rows.each_with_index do |row, i|
  # The first row i.e. i = 0 has the table headings which we ignore
  if i > 1
    # Each row has 3 columns which we store in cols [Team Name, Institution, Members]
    cols = row.css "td"
    team_name = cols[0].text
    college = cols[1].text
    # The members column has multiple <a> tags, one for each team member
    members = cols[2].css "a"

    # Get details of each member
    members.each do |member|
      # Get member username/handle from <a> tag
      handle = member.text
      # Some profile URLs have a space which needs to be encoded
      mem = member["href"].gsub " ", "%20"
      # Get profile page of member
      member_profile = Nokogiri::HTML(open("https://www.codechef.com" + mem))
      # Get real name of member
      name = member_profile.css(".user-name-box").text
      # Get country of member
      country = member_profile.css("span.user-country-name").text

      # Print details of member to stdout
      puts "#{team_name},\"#{college}\",#{country},#{handle},#{name}"
    end
  end
end
{% endhighlight %}

You'll notice I printed all the details instead of writing them to a file. This is because it's easier just to redirect the output of this script to a file from a terminal like this,

`./insomnia.rb 1 > insomnia1.csv`

Afterwards, I just combined the contents of each file to a single file like this,

`cat insomnia*.csv > all_insomnia.csv`

## Afterthought

There are improvements that can be made to this program.
Error handling is one, which I didn't bother with at the time since this was a one-off script.
However, one thing I should've done was handle timeouts.
The reason why I didn't just add a loop in the script to cover all 25 pages is that the connection would timeout due to flaky internet.
So instead I ran 25 instances of the script (one for each page) and restarted the ones which timed out.
I know it wasn't the best solution for the problem but it was the quickest to implement.

#### Footnotes

[^1]: Avishkar is MNNIT Allahabad's annual Techno-Management Festival
[^2]: Insomnia is a ACM-ICPC style programming contest and part of the CyberQuest category of events in Avishkar
