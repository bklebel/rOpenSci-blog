---
title: 'jstor: Parsing XML - Lessons From Writing a Package'
author: "Thomas Klebel"
date: "30 9 2018"
output: 
  html_document:
    keep_md: true
---



This is the story of `jstor`, a package that I wrote over the last year and a
half. It is a story about learning how to parse XML efficiently and creating a
piece of software that others will find useful. The story has many twists, and
I would like to take you with me – on a journey that has told me many lessons.

Before taking on the look back into the past, I want to give you some basic
information. JSTOR is a large archive for scientific texts, mainly known for 
their coverage of journal articles, although they recently added book chapters
and other sources as well. They make all of their content available for 
researchers to do text mining, citation analysis, and everything else you could
come up using metadata and data about the content of the articles. The package
I wrote, `jstor`, provides functions to import the metadata, and it has a few
helper functions for common cleaning tasks. Everything you need to know about 
how to use it is on the [package website](https://ropensci.github.io/jstor/).
You will find three vignettes, with a general introduction, examples on how to
import many files at once, and a few examples of known quirks of JSTORs data.
There is also a lengthy case study that also shows how to combine metadata and
data about the content of the articles. But for now let us turn back the clock
to follow along my journey of developing the package.

# Step 1: Hacking Away
Back in March 2017, I was starting out as a MA-student of
sociology in a research project
concerned with the scientific elite within sociology and economics. The project
had many goals, but writing an R package was not one of them. At the beginning
of my engagement,
I was presented with a dataset which was huge, at least for my terms: 
around 30GB of data, half of which was text, the other half 500,000 `.xml`-files.
The dataset was incredible in its depth: we basically sat on all articles
from JSTOR which belonged to the topics "sociology" and "economics". To repeat:
all articles that JSTOR has on those topics for all years.

My task was to somehow make this data accessible for our research. Since we are
sociologists and no computer experts and my knowledge of R was mainly 
self-taught, my approach was quite ad-hoc: "let's see, how we can extract 
relevant information for one file, and then maybe we can lapply over the whole
set of files." That is what the tidyverse philosophy and purrr tell you to do:
solve it for one case using a function, and apply this function to the whole
set of files, cases, nested rows, or whatever. Long story short, you can do it
like that, and I surely did it like that, but there would probably be more 
efficient solutions.

So, I had to start somewhere, and that was obviously importing the data into R.
After searching and trying out different packages, I settled on 
`xml2::read_xml()`. But then what. I had done a few pet projects with 
web-scraping, but had no knowledge of XPATH-expressions and how to access 
certain parts of the document directly. After some stumbling around, I had found
`xml2::as_list()` and was very happy: I could turn this unpleasant creature of
XML into a pretty list, which I was accustomed to. 
Since I tracked all my steps via git,
you can join in on my joy:

![](screenshots/lists.png)


Then I would use something
like `listviewer::jsonedit()` to inspect the elements, and extract what I 
needed. The approach was cumbersome, and the code was not pretty, since the
original documents are deeply nested and the structure is not always the same.
But it worked, and I was happy with it. 

My functions looked something like the following:

```r
extract_contributors <- function(contributors) {
  if (is.null(contributors)) {
    return(NA)
  } else {
    contributors %>%
      map(extract_name) %>%
      bind_rows() %>%
      rename(given_names = `given-names`) %>%
      mutate(author_number = 1:n())
  }
}


find_meta <- function(meta_list) {
  front <- meta_list$front
  article <- front$`article-meta`

  # authors
  contributors <- article$`contrib-group`
  contributors <- extract_contributors(contributors)

  # piece together all elements
  data_frame(
    journal_id = front$`journal-meta`$`journal-id`[[1]],
    article_id = article$`article-id`[[1]],
    article_title = article$`title-group`$`article-title`[[1]],
    authors = list(contributors),
    article_pages = extract_pages(article)
  )
}
```

The function would have been applied like so:

```r
# one file
file_path %>% 
  read_xml() %>% 
  as_list() %>% 
  find_meta()
  
# many files
file_paths %>% 
  map(read_xml) %>% 
  map(as_list) %>% 
  map(find_meta)
```



As can be expected when parsing deeply nested lists which do not always have
the same structure, this quickly escalated into
more and more complex and sometimes quite ridicoulus functions:

```r
extract_name <- function(x) {
  x %>%
    flatten() %>%
    flatten() %>%
    flatten() %>%
    .[!(names(.) %in% c("x", "addr-line"))] %>%
    as_data_frame()
}
```

But, as I said, I was solving problems with the tools I already knew, so I was
happy.

At some point though, I started benchmarking my solutions. I tried out 
`mclapply` and it doubled the speed of execution on my machine, but it was still
very slow: Parsing the content *after* importing and transforming it into a list
took 7.2 seconds for 3000 files. For 200,000 files of sociological 
articles this would amount to 8 hours. And importing and
converting to a list took a lot of time too. Although I didn't think at that 
time, that I would
need to import the files more than once or at most twice
(which I had to do later to update results
after we got new versions with better data), I decided that this was too slow,
and I looked for options. 

The first idea which I had was simply to scale computing power: I had a faster
machine at home, and planned to read the data into R at work, save it as `.rds`,
and then process it at home on the faster machine (for some reason I didn't want
to carry the original data home). Apparently, this is not as easy as it sounds.
If you parse a file with `xml2::read_xml()` you don't get the content of the
file, but only an object that points to the file, and you can then extract 
content with `xml_child()` and other functions.
But you cannot use `readr::write_rds` to save the file, instead you need
`xml2::xml_serialize()` and `xml2::xml_unserialize()` and they require an open
connection, and so on. I can tell you that, since Jim Hester kindly responded to [my
question on StackOverflow](https://stackoverflow.com/questions/44070577/write-xml-object-to-disk). 
He had a further hint:

> [...] I would seriously consider using XPATH expressions to extract the 
desired data, even if you have to rewrite code. [...] Xpath extracting just the elements you are interested in is going to run much faster than converting the entire data to a list first then manipulating that. ~Jim Hester


And this is what I did. For the fun of it, I benchmarked two versions: my 
function before re-writing to using XPATH expressions, and the function today.
The current version is a lot safer and handles many edge-cases, but it is still
around 5 times faster. Furthermore, it depends less on how complex the original
file is, and more on how much information is being read. With the new version
of the function, around 30% of the time is spent on reading the file, while 
with the old version it was around 85% of the time.


```r
microbenchmark::microbenchmark(
  convert = jst_example("sample_with_references.xml") %>% 
    read_xml() %>% 
    as_list(),
    
  old = jst_example("sample_with_references.xml") %>% 
    read_xml() %>% 
    as_list() %>% 
    find_meta(),
  
  new = jst_get_article(jst_example("sample_with_references.xml"))
)
```

```
## Unit: milliseconds
##     expr       min        lq     mean    median       uq       max neval
##  convert 35.621488 39.274647 44.29424 43.530006 46.64541 114.12766   100
##      old 42.038368 46.596185 51.49769 49.711072 52.43276 134.12999   100
##      new  8.675674  9.054154 10.33858  9.365577 10.16726  23.88457   100
```




<!---(I'd like to make the GitHub-repo public, where you could observe all of this 
beauty in greater detail, but I did what everybody tells you to never-ever do:
I hard-coded passwords for our server and commited them into git.)
--->



# Step two: doing it properly
Rewriting my functions was not that much of a hassle, in the end. I had turned
my functions into a package early on, and had already included many test cases
with [testthat](https://github.com/r-lib/testthat)
to make sure everything works as expected. This helped a lot for re-structuring
the code, since I already knew what my output should look like, and I only had 
to change the steps in between.

The work on re-writing progressed quickly. The first step was to simply extract
the identifier for the journal within which the article was published:

```r
find_meta <- function(xml_file) {
  if (identical("xml_document" %in% class(xml_file), FALSE)) {
    stop("Input must be a `xml_document`", call. = FALSE)
  }

  front <- xml_find_all(xml_file, "front")

  data_frame(
    journal_id = extract_jcode(front)
  )
}


extract_jcode <- function(front) {
  front %>%
    xml_child("journal-meta") %>%
    xml_find_first("journal-id") %>%
    xml_text()
}
```

I quickly added new parts, although it didn't feel all too easy: I basically had
to learn how to navigate within the document via XPATH. 

Another thing I took up
during this re-write was regularly benchmarking my functions. Since I lacked 
(and still lack) the technical expertise to know which approach would be faster,
I had to benchmark possibilities to figure it out. The following benchmark 
compares three versions of extracting the text from the node named `volume`.
Especially the first two seemed very similar to me and it was not at all 
obvious, which one would be faster.

```r
fun1 <- function() xml_find_all(test_file, "//volume/.//text()") %>% as.character()
fun2 <- function() xml_find_all(test_file, "//volume") %>% xml_text()
fun3 <- function() {
  xml_find_all(test_file, "front") %>%
    xml_child("article-meta") %>%
    xml_child("volume") %>%
    xml_text()
}

microbenchmark::microbenchmark(fun1(), fun2(), fun3(), times = 1000)
# Unit: microseconds
# expr     min       lq     mean   median       uq      max neval
# fun1() 267.684 290.5885 322.8992 301.4325 318.9060 2705.459  1000
# fun2() 233.723 250.3840 287.1126 263.1725 273.2180 2716.416  1000
# fun3() 686.542 725.0925 822.2103 763.4240 798.1155 3534.849  1000
```
Although the unit is microseconds here, the difference between `fun2` and 
`fun3` is quite substantial, if you want to do it 500,000 times: 4.5 minutes. If
you have to extract elements often (which I was doing), than this quickly adds 
up. What is the reason for this big difference?
I'd guess that `fun2` is fastest since it only calls two functions, and not 
four, and because `xml_text()` probably converts the content directly to 
character in C++, which is faster than calling `as.character` afterwards. But
I am still a *social* scientist, so I wouldn't bet on that explanation.


After rewriting and expanding the functionality, I was still not happy, however,
since the code was slower than I thought it should be. At this point,
I dug deeper again, using the package
[profvis](https://rstudio.github.io/profvis/) to find bottlenecks. Oddly
enough, I had introduced a bottleneck right at the beginning: I was using
`data_frame` (which is equivalent to `tibble()`) to create the object my 
function would return. Unfortunately, `tibble` (and `data.frame` as well) are
quite complex functions. They do type
checking and other things, and if you do this repeatedly, it quickly adds up and
is not very smart in general, since I know exactly what kind of data to expect
(if I wrote the rest of my functions appropriately).

You can see the difference yourself:

```r
microbenchmark::microbenchmark(
  tibble = tibble::tibble(a = rnorm(10), b = letters[1:10]),
  new_tibble = tibble::new_tibble(list(a = rnorm(10), b = letters[1:10]))
)
```

```
## Unit: microseconds
##        expr     min      lq     mean   median       uq      max neval
##      tibble 323.094 333.583 387.6665 338.8075 370.9175 1287.835   100
##  new_tibble  74.496  81.295 110.7023  86.2195  89.9845 1519.435   100
```

The difference is not huge (~50sec for 500000 documents), but my actual layout
was a lot more complex, since it was using nested structures in a weird way:


```r
# dummy function for getting the data
get_authors <- function() {
  tibble::tribble(
    ~first_name, ~last_name, ~author_number,
    "Pierre",    "Bourdieu", 1,
    "Max",       "Weber",    2,
    "Karl",      "Marx",     3
  )
}

# this approach made sense to me, since the data structure was indeed nested,
# and it seemed similar to the approach of mutate + map + unnest.
old_approach <- function() {
  res <- tibble::data_frame(id = "1",
                            authors = list(get_authors()))
  
  tidyr::unnest(res, authors)
}

# this is obviously a lot simpler
new_approach <- function() {
  data.frame(id = "1",
             authors = get_authors())
}


microbenchmark::microbenchmark(
  old = old_approach(),
  new = new_approach()
)
```

```
## Unit: milliseconds
##  expr      min        lq     mean   median       uq      max neval
##   old 5.566752 25.663635 47.80820 34.15654 51.17691 345.4876   100
##   new 1.839041  7.727445 14.64343 10.44802 15.97281 150.5579   100
```

The difference is significant, amounting to around 40 minutes more for
500,000 documents, if I would have kept the old version.

Unfortunately, using profvis for
such small and fast functions does not work well out of the box:


```r
profvis::profvis({
  old_approach()
})

#> Error in parse_rprof(prof_output, expr_source) : No parsing data available. 
#> Maybe your function was too fast?
```

A simple solution is to rerun the function many times. Note, however, that this
does not increase realiabity and reproducibility of the results. The function is
not measured 500 times, the result
being the mean or median (like in `microbenchmark`), but it is simply the 
aggregate of running the function 500 times. This can vary quite a bit. All in
all, tough, I found it still useful to judge if any part of the code is orders 
of magnitude slower than the rest.

In the following chunk
I define the function again for two reasons: first to separate all cumputations
into separate lines. This ensures, that we get a measurement for each line.
Second, the code needs to be supplied either within the call to profvis, or it
must be defined in a sourced file, otherwise the output will not be as 
informative, because profvis will not be able to access the functions' source
code.


```r
profvis::profvis({
  
  old_approach <- function(authors) {
    res <- tibble::data_frame(id = "1",
                              authors = list(authors))
    
    tidyr::unnest(res, authors)
  }

  authors <- get_authors()
  purrr::rerun(500, old_approach(authors))
})
```

![](screenshots/old_version.png)

We can see that the call to `tidyr::unnest` takes up a lot of time, thus
it would be good to get rid of it. The new approach does not need unnesting, and
is roughly 17 times faster:



```r
profvis::profvis({

  new_approach <- function(authors) {
    data.frame(id = "1",
               authors = authors)
  }
  
  authors <- get_authors()
  purrr::rerun(500, new_approach(authors))})
```

![](screenshots/new_version.png)

By assessing the efficiency of functions repeatedly and optimising several
parts,
I was able to trim down execution time considerably overall. For 25,000 files,
which is the maximum amount of files you can receive at one time through the
standard interface of JSTOR/DfR, execution time is slightly under 4 minutes, or
2 minutes if executed in parallel, at least on my moderately fast MacBook Pro.


# Lessons learned: 

- community is important
- what could I have done differently?
Decision for API: each function is self-contained. If I had extracted the parts
for import somehow, it would be faster to apply different functions (like for
metada and references and other stuff).

# Acknowledgements

# Options to contribute