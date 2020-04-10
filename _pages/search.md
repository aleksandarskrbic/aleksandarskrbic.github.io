---
title: "Search"
permalink: /search/
author_profile: true
header:
  image: "/images/search-header.jpg"
---

<!-- Html Elements for Search -->
<div id="search-container">
<input type="text" id="search-input" placeholder="Enter article name or tag(java, scala, akka, big data, kafka)">
<ul id="results-container"></ul>
</div>

<!-- Script pointing to search-script.js -->
<script src="../search-script.js" type="text/javascript"></script>

<!-- Configuration -->
<script>
SimpleJekyllSearch({
  searchInput: document.getElementById('search-input'),
  resultsContainer: document.getElementById('results-container'),
  json: '../search.json'
})
</script>