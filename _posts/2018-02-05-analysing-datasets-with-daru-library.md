---
layout: default
title:  "Introduction to analyzing datasets with daru library"
date:   2018-02-05 15:10:51 +0300
comments: true
tags: ruby
---

# Introduction to analyzing datasets with daru library

Though Ruby ecosystem is not very popular within the data-scientist community, when compared to languages like Python, R and Julia,
there is the [SciRuby](https://github.com/SciRuby) foundation, which provides the data science toolkit. One of the core libraries is
[daru](https://github.com/SciRuby/daru) - as states its
documentation, "daru is a library for storage, analysis, manipulation and visualization of data in Ruby". The library is
being actively developed and has drawn attention from data science community.

To get acquainted with daru, we can download some dataset and make manipulations with supplied data.

For the purposes of this article, I decided to stop on the [City of New York - Demographic statistics broken down by zip code](https://data.cityofnewyork.us/api/views/kku6-nxdu/rows.csv?accessType=DOWNLOAD) dataset,
available in different formats, but we'll go for the CSV version.

#### Getting to know the dataset

{% highlight ruby %}
require 'daru'

df = Daru::DataFrame.from_csv('Demographic_Statistics_By_Zip_Code.csv')
{% endhighlight %}

After typing this in pry console, you should see the output, which is basically the representation of a newly created
`Daru::DataFrame` object, containing dataset's data, but it's not of any help at its current state, so we should print put
available summary info of this dataframe:

{% highlight ruby %}
# Fetching number of dataframe vectors. In this case, we can imagine a dataframe as a table, with vectors being its columns
df.vectors.size

=> 46

# Fetching number of rows
df.nrows

=> 236

# Fetching particular row, 0 to 235 available for this dataframe
# The row is truncated to first 15 entries.

df.row_at(4)

=> #<Daru::Vector(46)>
    JURISDICTION NAME                10005
   COUNT PARTICIPANTS                    2
         COUNT FEMALE                    2
       PERCENT FEMALE                    1
           COUNT MALE                    0
         PERCENT MALE                    0
 COUNT GENDER UNKNOWN                    0
 PERCENT GENDER UNKNO                    0
   COUNT GENDER TOTAL                    2
 PERCENT GENDER TOTAL                  100
 COUNT PACIFIC ISLAND                    0
 PERCENT PACIFIC ISLA                    0
 COUNT HISPANIC LATIN                    0
 PERCENT HISPANIC LAT                    0
 COUNT AMERICAN INDIA                    0
                  ...                  ...

df.vectors

=> #<Daru::Index(46): {JURISDICTION NAME, COUNT PARTICIPANTS, COUNT FEMALE, PERCENT FEMALE, COUNT MALE, PERCENT MALE,
  COUNT GENDER UNKNOWN, PERCENT GENDER UNKNOWN, COUNT GENDER TOTAL, PERCENT GENDER TOTAL, COUNT PACIFIC ISLANDER,
  PERCENT PACIFIC ISLANDER, COUNT HISPANIC LATINO, PERCENT HISPANIC LATINO, COUNT AMERICAN INDIAN,
  PERCENT AMERICAN INDIAN, COUNT ASIAN NON HISPANIC, PERCENT ASIAN NON HISPANIC, COUNT WHITE NON HISPANIC,
  PERCENT WHITE NON HISPANIC ... PERCENT PUBLIC ASSISTANCE TOTAL}>

# fetching particular column

df['JURISDICTION NAME']
=> #<Daru::Vector(236)>
                   JURISDICTION NAME
                 0             10001
                 1             10002
                 2             10003
                 3             10004
                 4             10005
                 5             10006
                 6             10007
                 7             10009
                 8             10010
                 9             10011
                10             10012
                11             10013
                12             10014
                13             10016
                14             10017
               ...               ...
{% endhighlight %}

Now we can already make some sense of this dataset. Each row contains data for jurisdictions determined by zip code.

Getting deeper into summarizing dataset:

{% highlight ruby %}
puts df.summary

Number of rows: 236
  Element:[COUNT FEMALE]
  == COUNT FEMALE
    n :236
    non-missing:236
    median: 0.0
    mean: 10.2966
    std.dev.: 28.1891
    std.err.: 1.8350
    skew: 4.4564
    kurtosis: 22.4802

# ...
# rest of the output is omitted
{% endhighlight %}

This command prints out summary information for each column of the dataframe, with columns breakdown.

`n` indicates a total number of values
`non-missing` is the number of values that are actually present for given column. 236 means that every row has COUNT FEMALE filled.
The `median`, `mean`, `std.dev`, `std.err`, `skew` and `kurtosis` are referring to the most common idioms from the statistical computation.

We also can summarize dataframe partially, using `head` and `tail` methods. These two are slicing data frame respectively from
its beginning(head) or from its end(tail) accounting only 10 rows by default, and therefore only 10 rows would be summarized.

{% highlight ruby %}
  Element:[COUNT FEMALE]
  ==
    n :10
    non-missing:10
    median: 1.5
    mean: 4.8000
    std.dev.: 8.3506
    std.err.: 2.6407
    skew: 1.2688
    kurtosis: -0.3123
{% endhighlight %}

Now let's proceed to actually plot some data and get the taste of the visualization using daru.

#### Visualization

The visualization part by now supports three backends, the default [nyaplot](https://github.com/domitry/nyaplot)
which can be optionally changed to [gnuplotrb](https://github.com/SciRuby/gnuplotrb) or [gruff](https://github.com/topfunky/gruff)

The most obvious task is to make bar chart containing top 10 jurisdictions with US citizens.

To express our intents more clearly, we can slice our dataset to leave only needed columns:

{% highlight ruby %}
wf = df['JURISDICTION NAME', 'COUNT US CITIZEN']

=> #<Daru::DataFrame(236x2)>

# ...

wf = wf.sort(['COUNT US CITIZEN']).tail

=> #<Daru::DataFrame(10x2)>
            JURISDICTI COUNT US C
        119      11218        102
        124      11223        102
        210      12428        124
        222      12754        133
        229      12783        197
        120      11219        212
        228      12779        241
        130      11230        245
        218      12734        252
        232      12789        271

# Here I had to make a new dataframe because 'JURISDICTION NAME' vector is treated as a numeric otherwise

data = Daru::DataFrame.new(x: wf['JURISDICTION NAME'].map(&:to_s), y: wf['COUNT US CITIZEN'])

plot = dt.plot(type: :bar, x: :x, y: :y) do |plot, _diag|
  plot.x_label("Jurisdictions")
  plot.y_label("US citizen count")
end

plot.export_html

{% endhighlight %}

And the html exported plot should appear in the working directory. It is advised to work with jupyter notebooks
for plotting tasks, but the installation of those tools is a separate story and we use `export_html` way for the simplicity.

![alt text](https://github.com/Shkrt/shkrt.github.io/raw/master/img/plot_1.png "Output example")

As you might have noticed, extracting, summarizing and manipulating data with daru library is quite simple even you don't come from
data science world, and even simpler if you are already used to work with specialized tools like R, Octave, and others.

Suggested reading:

[Daru documentation](https://github.com/SciRuby/daru)

