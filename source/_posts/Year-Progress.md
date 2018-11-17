---
title: Year Progress
date: 2018-11-08 11:49:31
tags:
top: true
---

{% centerquote %}
<div id="yearglass-web"></div>
{% endcenterquote %}

<script>
	const today = new Date();
	const year = today.getFullYear();
	const thisYear = new Date(year, 0, 1);
	const nextYear = new Date(year + 1, 0, 1);
	const oneDay = today.getMilliseconds();
	const passed = Math.floor((today - thisYear) / oneDay);
	const total = Math.floor((nextYear - thisYear) / oneDay);
	const percentage = passed / total;
	const space = 15;

	function repeat(s, n) {
		return new Array(Math.floor(n + 1)).join(s);
	}

	document.getElementById("yearglass-web").innerHTML = repeat("▓▓", space * percentage) + repeat("░░", space * (1 - percentage)) + " " + Math.floor(percentage * 100) + "%";
</script>
