## ▒ μFuzzy

A tiny, efficient, fuzzy search that doesn't suck.
This is my fuzzy 🐈. [There are many like it](#a-biased-appraisal-of-similar-work), but this one is mine.

---
### Overview

uFuzzy is a [fuzzy search](https://en.wikipedia.org/wiki/Approximate_string_matching) library designed to match a relatively short search phrase (needle) against a large list of short-to-medium phrases (haystack).
It might be best described as a more forgiving [String.indexOf()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/indexOf).
Common use cases are list filtering, auto-complete/suggest, and title/name/description/filename/function searches.

Each uFuzzy match must contain all alpha-numeric characters from the needle in the same sequence, so is likely a poor fit for applications like spellcheck or fulltext/document search.
However, its speed leaves ample headroom to [match out-of-order terms](https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy&outOfOrder&search=spac%20ca) by combining results from all permutations of the needle.
When held _just right_, it can efficiently match against multiple object properties, too.

---
### Features

- **Junk-free, high quality results** that are _dataset-independent_. No need to fine-tune indexing options or boosting params to attain some arbitrary quality score cut-off.
- **Straightforward fuzziness control** that can be explained to your grandma in 5min.
- **Sorting you can reason about** and customize using a simple `Array.sort()` which gets access to each match's stats/counters. There's no composite, black box "score" to understand.
- **Concise set of options** that don't interact in mysterious ways to drastically alter combined behavior.
- **Fast with low resource usage** - there's no index to build, so startup is below 1ms with near-zero memory overhead. Searching a three-term phrase in a 162,000 phrase dataset takes 12ms with in-order terms or 50ms with out-of-order terms.
- **Micro, with zero dependencies** - currently [< 4KB min](https://github.com/leeoniya/uFuzzy/blob/main/dist/uFuzzy.iife.min.js)

[![uFuzzy demo](uFuzzy.png)](https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy&outOfOrder&search=spac%20ca)

---
### Demos

**NOTE:** The [testdata.json](https://github.com/leeoniya/uFuzzy/blob/main/demos/testdata.json) file is a diverse 162,000 string/phrase dataset 4MB in size, so first load may be slow due to network transfer.
Try refreshing once it's been cached by your browser.

First, uFuzzy in isolation to demonstrate its performance.

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy&search=super%20ma

Now the same comparison page, booted with [fuzzysort](https://github.com/farzher/fuzzysort), [QuickScore](https://fwextensions.github.io/quick-score-demo/), and [Fuse.js](https://fusejs.io/):

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,fuzzysort,QuickScore,Fuse&search=super%20ma

Here is the full library list but with a reduced dataset (just `hearthstone_750`, `urls_and_titles_600`) to avoid crashing your browser:

https://leeoniya.github.io/uFuzzy/demos/compare.html?lists=hearthstone_750,urls_and_titles_600&search=moo

---
### Installation

### Node

```
npm i @leeoniya/ufuzzy
```

```js
const uFuzzy = require('@leeoniya/ufuzzy');
```

### Browser

```js
<script src="./dist/uFuzzy.iife.min.js"></script>
```

---
### Usage

uFuzzy works in 3 phases:

1. **Filter** - This filters the full `haystack` with a fast RegExp compiled from your `needle` without doing any extra ops. It returns an array of matched indices in original order.
2. **Info** - This collects more detailed stats about the filtered matches, such as start offsets, fuzz level, prefix/suffix counters, etc. It also gathers substring match positions for range highlighting. Finally, it filters out any matches that don't conform to the desired prefix/suffix rules. To do all this it re-compiles the `needle` into two more-expensive RegExps that can partition each match. Therefore, it should be run on a reduced subset of the haystack, usually returned by the Filter phase. The [uFuzzy demo](https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy) is gated at <= 1,000 filtered items, before moving ahead with this phase.
3. **Sort** - This does an `Array.sort()` to determine final result order, utilizing the `info` object returned from the previous phase. A custom sort function can be provided via a uFuzzy option: `{sort: (info, haystack, needle) => idxsOrder}`.

```js
let haystack = [
    'puzzle',
    'Super Awesome Thing (now with stuff!)',
    'FileName.js',
    '/feeding/the/catPic.jpg',
];

let needle = 'feed cat';

let opts = {};

let uf = new uFuzzy(opts);

// pre-filter
let idxs = uf.filter(haystack, needle);

// sort/rank only when <= 1,000 items
if (idxs.length <= 1e3) {
  let info = uf.info(idxs, haystack, needle);

  // order is a double-indirection array (a re-order of the passed-in idxs)
  // this allows corresponding info to be grabbed directly by idx, if needed
  let order = uf.sort(info, haystack, needle);

  // render post-filtered & ordered matches
  for (let i = 0; i < order.length; i++) {
    // using info.idx here instead of idxs because uf.info() may have
    // further reduced the initial idxs based on prefix/suffix rules
    console.log(haystack[info.idx[order[i]]]);
  }
}
else {
  // render pre-filtered but unordered matches
  for (let i = 0; i < idxs.length; i++) {
    console.log(haystack[i]);
  }
}
```

---
### API

A liberally-commented 100 LoC [uFuzzy.d.ts](https://github.com/leeoniya/uFuzzy/blob/main/dist/uFuzzy.d.ts) file.

---
### Options

Options with an **inter** prefix apply to allowances _in between_ search terms, while those with an **intra** prefix apply to allowances _within_ each search term.

<table>
    <thead>
        <tr>
            <th>Option</th>
            <th>Description</th>
            <th>Default</th>
            <th>Examples</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><code>intraIns</code></td>
            <td>Max number of extra chars allowed<br>between each char within a term</td>
            <td><code>0</code></td>
            <td>
                Searching "cat"...<br>
                <code>0</code> can match: <b>cat</b>, s<b>cat</b>, <b>cat</b>ch, va<b>cat</b>e<br>
                <code>1</code> also matches: <b>ca</b>r<b>t</b>, <b>c</b>h<b>a</b>p<b>t</b>er, out<b>ca</b>s<b>t</b><br>
            </td>
        </tr>
        <tr>
            <td><code>interIns</code></td>
            <td>Max number of extra chars allowed between terms</td>
            <td><code>Infinity</code></td>
            <td>
                Searching "where is"...<br>
                <code>Infinity</code> can match: <b>where is</b>, <b>where</b> have blah w<b>is</b>dom<br>
                <code>5</code> cannot match: where have blah wisdom<br>
            </td>
        </tr>
        <tr>
            <td><code>intraChars</code></td>
            <td>Partial regexp for allowed extra<br>chars between each char within a term</td>
            <td><code>[a-z\d]</code></td>
            <td>
                <code>[a-z\d]</code> matches only alpha-numeric (case-insensitive)<br>
                <code>[\w-]</code> would match alpha-numeric, undercore, and hyphen<br>
            </td>
        </tr>
        <tr>
            <td><code>intraFilt</code></td>
            <td>Callback for excluding results based on term &amp; match</td>
            <td><code>(term, match, index) => true</code></td>
            <td>
                Do your own thing, maybe...
                - Length diff threshold<br>
                - Levenshtein distance<br>
                - Term offset or content<br>
            </td>
        </tr>
        <tr>
            <td><code>interChars</code></td>
            <td>Partial regexp for allowed chars between terms</td>
            <td><code>.</code></td>
            <td>
                <code>.</code> matches all chars<br>
                <code>[^a-z\d]</code> would only match whitespace and punctuation<br>
            </td>
        </tr>
        <tr>
            <td><code>interLft</code></td>
            <td>Determines allowable term left boundary</td>
            <td><code>0</code></td>
            <td>
                Searching "mania"...<br>
                <code>0</code> any - anywhere: ro<b>mania</b>n<br>
                <code>1</code> loose - whitespace, punctuation, alpha-num, case-change transitions: Track<b>Mania</b>, <b>mania</b>c<br>
                <code>2</code> strict - whitespace, punctuation: <b>mania</b>cally<br>
            </td>
        </tr>
        <tr>
            <td><code>interRgt</code></td>
            <td>Determines allowable term right boundary</td>
            <td><code>0</code></td>
            <td>
                Searching "mania"...<br>
                <code>0</code> any - anywhere: ro<b>mania</b>n<br>
                <code>1</code> loose - whitespace, punctuation, alpha-num, case-change transitions: <b>Mania</b>Star<br>
                <code>2</code> strict - whitespace, punctuation: <b>mania</b>_foo<br>
            </td>
        </tr>
        <tr>
            <td><code>sort</code></td>
            <td>Custom result sorting function</td>
            <td><code>(info, haystack, needle) => idxsOrder</code></td>
            <td>
                Default: <a href="https://github.com/leeoniya/uFuzzy/blob/bba02537334ae9d02440b86262fbfa40d86daa54/src/uFuzzy.js#L32-L52">Search sort</a>, prioritizes full term matches and char density<br>
                Demo: <a href="https://github.com/leeoniya/uFuzzy/blob/bba02537334ae9d02440b86262fbfa40d86daa54/demos/compare.html#L264-L288">Typeahead sort</a>, prioritizes start offset and match length<br>
            </td>
        </tr>
    </tbody>
</table>

---
### A biased appraisal of similar work

This assessment is extremely narrow and, of course, biased towards my use cases, text corpus, and my complete expertise in operating my own library.
It is highly probable that I'm not taking full advantage of some feature in other libraries that may significantly improve outcomes along some axis;
I welcome improvement PRs from anyone with deeper library knowledge than afforded by my hasty 10min skim over any "Basic usage" example and README doc.

#### Search quality

Can-of-worms #1.

Before we discuss [performance](#performance) let's talk about search quality, because speed is irrelevant when your results are a strange medly of "Oh yeah!" and "WTF?".

Search quality is very subjective.
What constitutes a good top match in a "typeahead / auto-suggest" case can be a poor match in a "search / find-all" scenario.
Some solutions optimize for the latter, some for the former.
It's common to find knobs that skew the results in either direction, but these are often by-feel and imperfect, being little more than a proxy to producing a single, composite match "score".

Let's take a look at some matches produced by the most popular fuzzy search library, [Fuse.js](https://github.com/krisk/Fuse) and some others for which match highlighting is implemented in the demo.

Searching for the partial term **"twili"**, we see these results appearing above numerous obvious **"twilight"** results:

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,fuzzysort,QuickScore,Fuse&search=twili

- **twi**r**li**ng
- **T**he total number of received alerts that **w**ere **i**nva**li**d.
- **T**om Clancy's Ghost Recon **Wil**dlands - AS**I**A Pre-order Standard Uplay Activation
- **t**heHunter™: Call of the **Wi**ld - Bearclaw **Li**te CB-60

Not only are these poor matches in isolation, but they actually rank higher than literal substrings.

Finishing the search term to **"twilight"**, _still_ scores bizzare results higher:

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,fuzzysort,QuickScore,Fuse&search=twilight

- Magic: **T**he Gathering - Duels of the Planeswalkers **Wi**ngs of **Light** Unlock
- **T**he **Wil**d E**ight**

Some engines do better with partial prefix matches, at the expense of higher startup/indexing cost:

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,FlexSearch,match-sorter,MiniSearch&search=twili

Here, `match-sorter` returns 1,384 results, but only the first 40 are relevant. How do we know where the cut-off is?

https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,FlexSearch,match-sorter,MiniSearch&search=super

<!--
twil  0.1683 ok, 0.25+ bad
chest 0.1959 ok, 0.2+ bad
train
nin tur
puzz, puzl (MiniSearch, {fuzzy: 0.4}, uFuzzy, intraIns: 1)
-->

#### Performance

Can-of-worms #2.

All benchmarks suck, but this one might suck more than others.

- I've tried to follow any "best performance" advice when I could find it in each library's docs, but it's a certainty that some stones were left unturned when implementing ~20 different search engines.
- Despite my best efforts, result quality is still extremely variable between libraries, and even between search terms. In some cases, results are very poor but the library is very fast; in other cases, the results are better, but the library is quite slow. What use is extreme speed when the search quality is sub-par? This is a subjective, nuanced topic that will surely affect how you interpret these numbers. I consider uFuzzy's search quality second-to-none, so my view of most faster libraries is typically one of quality trade-offs I'm happy not to have made. I encourage you to evaluate the results for all benched search phrases manually to decide this for yourself.
- Many fulltext & document-search libraries compared here are designed to work best with exact terms rather than partial matches (which this benchmark is skewed towards).

Still, something is better than a hand-wavy YMMV/do-it-yourself dismissal and certainly better than nothing.

#### Benchmark

- Each benchmark can be run by changing the `libs` parameter to the desired library name: https://leeoniya.github.io/uFuzzy/demos/compare.html?bench&libs=uFuzzy
- Results output is suppressed in `bench` mode to avoid benchmarking the DOM.
- Measurements are taken in the Performance secrion of Chrome's DevTools by recording several reloads of the bench page, with forced garbage collection in between. The middle/typical run is used to collect numbers.
- The search corpus is 162,000 words and phrases, loaded from a 4MB [testdata.json](https://github.com/leeoniya/uFuzzy/blob/main/demos/testdata.json).
- The benchmark types and then deletes, character-by-character (every 100ms) the following search terms, triggering a search for each keypress: `test`, `chest`, `super ma`, `mania`, `puzz`, `prom rem stor`, `twil`.

To evaluate the results for each library, or to compare several, simply visit the same page with more `libs` and without `bench`: https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy,fuzzysort,QuickScore,Fuse&search=super%20ma.

![profile example](bench.png)

There are several metrics evaluated:

- Init time - how long it takes to load the library and build any required index to perform searching.
- Bench runtime - how long it takes to execute all searches.
- Memory required - peak JS heap size used during the bench as well as how much is still retained after a forced garbage collection at the end.
- GC cost - how much time is needed to collect garbage at the end (main thread jank)

<!--
https://bestofjs.org/projects?tags=search
-->

<table>
    <thead>
        <tr>
            <th>Lib</th>
            <th>Stars</th>
            <th>Size (min)</th>
            <th>Init</th>
            <th>Search</th>
            <th>Heap (peak)</th>
            <th>Retained</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>
                <a href="https://github.com/leeoniya/uFuzzy">uFuzzy</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=uFuzzy&search=super%20ma">try</a>)
            </td>
            <td>★ 0</td>
            <td>4KB</td>
            <td>0.3ms</td>
            <td>620ms</td>
            <td>25.5MB</td>
            <td>7.5MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/krisk/Fuse">Fuse.js</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=Fuse&search=super%20ma">try</a>)
            </td>
            <td>★ 14.8k</td>
            <td>23.5KB</td>
            <td>40ms</td>
            <td>35432ms</td>
            <td>372MB</td>
            <td>14MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/nextapps-de/flexsearch">FlexSearch (Light)</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=FlexSearch&search=super%20ma">try</a>)
            </td>
            <td>★ 8.9k</td>
            <td>5.9KB</td>
            <td>3600ms</td>
            <td>130ms</td>
            <td>673MB</td>
            <td>316MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/olivernn/lunr.js">Lunr.js</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=Lunr&search=super%20ma">try</a>)
            </td>
            <td>★ 8.2k</td>
            <td>29.4KB</td>
            <td>1900ms</td>
            <td>755ms</td>
            <td>450MB</td>
            <td>231MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/LyraSearch/lyra">LyraSearch</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=LyraSearch&search=super%20ma">try</a>)
            </td>
            <td>★ 3.3k</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/kentcdodds/match-sorter">match-sorter</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=match-sorter&search=super%20ma">try</a>)
            </td>
            <td>★ 3.1k</td>
            <td>7.3KB</td>
            <td>0.03ms</td>
            <td>8900ms</td>
            <td>73.4MB</td>
            <td>7.4MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/farzher/fuzzysort">fuzzysort</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=fuzzysort&search=super%20ma">try</a>)
            </td>
            <td>★ 3k</td>
            <td>5.5KB</td>
            <td>50ms</td>
            <td>1400ms</td>
            <td>176MB</td>
            <td>84MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/kbrsh/wade">Wade</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=Wade&search=super%20ma">try</a>)
            </td>
            <td>★ 3k</td>
            <td>4KB</td>
            <td>770ms</td>
            <td>315ms</td>
            <td>436MB</td>
            <td>42MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/bevacqua/fuzzysearch">fuzzysearch</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=fuzzysearch&search=super%20ma">try</a>)
            </td>
            <td>★ 2.6k</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/weixsong/elasticlunr.js">Elasticlunr.js</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=Elasticlunr&search=super%20ma">try</a>)
            </td>
            <td>★ 1.9k</td>
            <td>18.1KB</td>
            <td>1000ms</td>
            <td>1630ms</td>
            <td>224MB</td>
            <td>70MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/lucaong/minisearch">MiniSearch</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=MiniSearch&search=super%20ma">try</a>)
            </td>
            <td>★ 1.5k</td>
            <td>22.4KB</td>
            <td>550ms</td>
            <td>1900ms</td>
            <td>423MB</td>
            <td>64MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/Glench/fuzzyset.js">Fuzzyset</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=Fuzzyset&search=super%20ma">try</a>)
            </td>
            <td>★ 1.3k</td>
            <td>2.8KB</td>
            <td>3100ms</td>
            <td>850ms</td>
            <td>660MB</td>
            <td>238MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/fergiemcdowall/search-index">search-index</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=search-index&search=super%20ma">try</a>)
            </td>
            <td>★ 1.3k</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/rmm5t/liquidmetal">LiquidMetal</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=LiquidMetal&search=super%20ma">try</a>)
            </td>
            <td>★ 285</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/itemsapi/itemsjs">ItemJS</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=ItemJS&search=super%20ma">try</a>)
            </td>
            <td>★ 260</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/wouter2203/fuzzy-search">FuzzySearch</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=fuzzy-search&search=super%20ma">try</a>)
            </td>
            <td>★ 184</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/jeancroy/FuzzySearch">FuzzySearch2</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=FuzzySearch2&search=super%20ma">try</a>)
            </td>
            <td>★ 173</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/fwextensions/quick-score">QuickScore</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=QuickScore&search=super%20ma">try</a>)
            </td>
            <td>★ 131</td>
            <td>9.1KB</td>
            <td>26ms</td>
            <td>7100ms</td>
            <td>171MB</td>
            <td>12.3MB</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/jhawthorn/fzy.js/">fzy</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=fzy&search=super%20ma">try</a>)
            </td>
            <td>★ 115</td>
        </tr>
        <tr>
            <td>
                <a href="https://github.com/grafana/grafana/blob/main/packages/grafana-ui/src/utils/fuzzy.ts">fuzzyMatch</a>
                (<a href="https://leeoniya.github.io/uFuzzy/demos/compare.html?libs=fuzzyMatch&search=super%20ma">try</a>)
            </td>
            <td>★ 0</td>
        </tr>
    </tbody>
</table>