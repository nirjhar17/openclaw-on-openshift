---
layout: default
---

{% capture blog %}{% include_relative deploying-openclaw-on-openshift.md %}{% endcapture %}
{{ blog | markdownify }}
