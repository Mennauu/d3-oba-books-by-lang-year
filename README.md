# Interactive data visualization in D3

**A visualization that shows the total amount of books from the OBA per language, with the possibility to filter the results per publication year**

*All books*
![chart](/images/chart.png)

*All books from a specific publication year*
![chart adjusted](/images/chart-adjusted.png)

## Table of Contents

- [Description](#description)
- [Data](#data)
  - [Retrieve](#retrieve)
  - [Filter](#filter)
  - [Update](#update)
- [Interaction](#interaction)
- [Code changes](#code-changes)
- [Features](#features)
- [Credits](#credits)
- [Sources](#sources)
- [License](#license)

## Description
The idea behind the datavisualization is to see how many books the OBA got in a specific language, which is why all languages are included even if there is only one book of that particular language.

## Data

The data was taken from the Amsterdam Public Library (Openbare Bibliotheek Amsterdam, OBA). You can find information about the API here: [Aquabrowser API Documentation](https://zoeken.oba.nl/api/v1/)

### Retrieve

To retrieve the data I used Gijs Laarman's OBA Scraper. His tool scrapes the OBA API and transforms XML to JSON. You can find more about his OBA Scraper here: [@gijslaarman/oba-scraper](https://www.npmjs.com/package/@gijslaarman/oba-scraper)

I wanted to retrieve ALL (yes, ALL) books with two variables:
* Publication year
* Language

So I ran the script, retrieving 20.000 results per time. This resulted in 22 files because there are around 440.000 books in total. I combined all the files to one big JSON file. Underneath a snippet from the data.

```json
{"pubYear":"2012","language":"dut"},
{"pubYear":"1996","language":"ger"},
{"pubYear":"2013","language":"eng"},
{"pubYear":"1996","language":"dut"}
```

### Filter

Now it's time to filter the data. This took most of the time, so I'll explain this the most. I was struggling, until [Tim Ruiterkamp](https://github.com/timruiterkamp) gave a little presentation about nesting in D3. That was exactly what I needed! A gift from heaven. He explained how nesting worked and after the presentation I looked it up online. I found two websites which explained it really well; http://learnjsdata.com/group_data.html and http://bl.ocks.org/phoebebright/raw/3176159/. It was still a lot of thinking and quite difficult to get exactly what I wanted, though.

I wanted to filter the data where all languages with the same publication year were combined, and count the amount of times this happened per language.


```javascript
d3.json("results.json", function(json) {
  let data = d3.nest()
  .key(function(d) { return d.pubYear }).sortKeys(d3.ascending)
  .key(function(d) { return d.language })
  .rollup(function (v) { return v.length })
  .entries(json)
  .map(function(d) { return {
    year: d.key,
    languages: d.values

    .map(function(v) { return {
      lang: v.key,
      amount: v.value
    }})
  }})
})
```

Underneath a snippet from the filtered data.
```javascript
{ year: "1939", languages: (9) […] }

// Nested in Languages
{ lang: "dut", amount: 375 }
{ lang: "null", amount: 81 }
{ lang: "ger", amount: 45 }
{ lang: "eng", amount: 76 }
{ lang: "fre", amount: 28 }
{ lang: "afr", amount: 2 }
{ lang: "dan", amount: 1 }
{ lang: "grc", amount: 3 }
{ lang: "fri", amount: 1 }
```

But, we aren't done yet. Because we also want the TOTAL amount of books per language. HOW ON EARTH are we going to do that. I was eager to figure this out myself. Which I did, but I think it could be done way easier.

```javascript
// Loop through languages
const allLanguagesAndAmount = data.map(function(d) { return d.languages })

// Get all languages and amounts per object
const formatLanguagesAndAmount = [...new Set([].concat(...allLanguagesAndAmount))]

const lang = {}
const amountPerLanguage = []

// format all objects into one object per language and sum up the amount
formatLanguagesAndAmount.forEach(function(d) {
  if (!lang[d.lang]) { return lang[d.lang] = d }

  return lang[d.lang].amount = lang[d.lang].amount + d.amount
})

// Push the keys in an array
d3.keys(lang).map(function(key) {
  amountPerLanguage.push(lang[key])
})
```

Last but not least, we sort the data from high to low based on amount

```javascript
amountPerLanguage.sort(function(a, b) {
  return b.amount-a.amount
})
```

THE RESULT

```javascript
{ lang: "dut", amount: 283667 }
{ lang: "eng", amount: 85724 }
{ lang: "null", amount: 30920 }
{ lang: "ger", amount: 19371 }
{ lang: "fre", amount: 11293 }
{ lang: "spa", amount: 3090 }
```

BTW, you can see one language as **null**. Null stands for all the books that were not given any language. *Shame on you OBA ;-)*.

### Update

We've got the data, we drew the chart. Now we have to update it with new data based on the selected year.

1. We first get the value from the selected option and pass it to the update function

```javascript
// Trigger the update function
d3.select("#choose-year").on("change", function() {
  const year = d3.select(this).property("value")
  update(data, year)
})
```

2. Then we retrieve the data where the year is the same as selected year and afterwards we loop over it to get the data

```javascript
// Retrieve data where year is the same as selected year
const dataBasedOnChosenYear = data.filter(function(d) { return d.year === year })

// Then loop over it to get the data
const languagesAndAmount = dataBasedOnChosenYear.map(function(d) { return d.languages })
```

3. All that's left is to set the new xAxis and yAxis, and afterwards replace or adjust the bars and the amounts.

```javascript
// scale the range of the new data
x.domain(languagesAndAmount[0].map(function (d) { return d.lang }))
y.domain([0, d3.max(languagesAndAmount[0], function(d) { return d.amount })]).nice()
```

## Interaction
You can interact with the visualization by choosing a year. After choosing a year, the data belonging to that year will be displayed in the chart. You can reset it back to ALL books by clicking on the "Reset" button. I'll be honest, this isn't done the right way (by resetting data in the chart). It currently just refreshes the page. It's on my to-do list.

## Code changes

I used code from the answer from the question asked on this stackoverflow post: [How to merge objects and sum just some values of duplicate objects?](https://stackoverflow.com/questions/45735910/how-to-merge-objects-and-sum-just-some-values-of-duplicate-objects)
It was exactly what I needed to accomplish what I wanted.

*Original code*

```javascript
var o = {};

a.forEach((i) => {
  var id = i.id;
  i.points = parseInt(i.points)
  if (!o[id]) {
    return o[id] = i
  }
  return o[id].points = o[id].points + i.points
})

var a2 = []
Object.keys(o).forEach((key) => {
 a2.push(o[key])
})
```

*Code in my project*
```javascript
const lang = {}
const amountPerLanguage = []

// format all objects into one object per language and sum up the amount
formatLanguagesAndAmount.forEach(function(d) {
  if (!lang[d.lang]) { return lang[d.lang] = d }

  return lang[d.lang].amount = lang[d.lang].amount + d.amount
})

// Push the keys in an array
d3.keys(lang).map(function(key) {
  amountPerLanguage.push(lang[key])
})
```

*I also used the select form written by rlemon, which you can find here: [Is it possible to use a for loop in select in HTML and how?](https://stackoverflow.com/questions/12725265/is-it-possible-to-use-a-for-loop-in-select-in-html-and-how?answertab=votes#tab-top). I don't want to go into much detail about this. I didn't write it myself because I knew about the code that he had written because I used it before in a different project, and wanted to spend my scarce time on D3 related features. I'm not reinventing the wheel. The only adjustment I made is getting the results in reversed order.*

## D3 Features
These are all the D3 features I used in this project. You can read about them on D3's Github: https://github.com/d3

```javascript
d3.json()
d3.nest()
d3.select()
d3.keys()
d3.max()
d3.scaleOrdinal()
d3.scaleBand()
d3.scaleLinear()
d3.axisBottom()
d3.axisLeft()
d3.format()
```

## Fails

I first wanted to show the amount of books per language per year on a (heat)map. Of course, you can't really do that. They might speak a language in multiple countries. Thanks for pointing that out during our feedback meeting, Laurens.

This is what I had already made before the feedback, which is fine because I can still use it for future projects. All countries are separate SVG's

![map](/images/map.png)

After that I made a dynamic grouped bar chart which was based on the xAsis having years. It took me quite some time to figure out how to make it dynamic. In the end I could just add a country and amounts per year in the data file and it would add a new bar to the group for each year. Well, that was a big waste of time, lol. I didn't need that at all in the chart that I was going to make based off the idea I had. I still thought it was cool though, because I did manage to pull it off.

![grouped bars](/images/grouped-bars.png)

___
## To-do list
- [ ] Reset button to initial data based on chart reset, not on page reload.
- [ ] The amounts on bars are sometimes buggy when choosing different years.
- [ ] Add more filter functions, like choosing between multiple years, or on genre.
- [ ] Clean the code (and rewrite to ES6)
___

## Credits

I would like to thank [Gijs Laarman](https://github.com/gijslaarman) for his OBA-Scraper and [Tim Ruiterkamp](https://github.com/timruiterkamp) for his presentation on nesting.

## Sources

These are some of the sources that gave me insights on some things, like nesting. I didn't "directly" copy any of the code, but they helped in a way. I tried to write my own code as much as possible for this project, as a learning curve.

* https://bl.ocks.org/mbostock/3887051
* https://stackoverflow.com/questions/2917175/return-multiple-values-in-javascript
* https://stackoverflow.com/questions/45201367/d3-data-function-not-working-as-expected-multiple-data-sources
* http://jsbin.com/nuyipikaye/edit?js,output
* https://bl.ocks.org/bricedev/0d95074b6d83a77dc3ad
* https://stackoverflow.com/questions/37172184/rename-key-and-values-in-d3-nest
* https://stackoverflow.com/questions/11922383/access-process-nested-objects-arrays-or-json
* https://stackoverflow.com/questions/8085004/iterate-through-nested-javascript-objects
* https://stackoverflow.com/questions/11189284/d3-axis-labeling
* http://learnjsdata.com/group_data.html
* http://bl.ocks.org/phoebebright/raw/3176159/

## License
