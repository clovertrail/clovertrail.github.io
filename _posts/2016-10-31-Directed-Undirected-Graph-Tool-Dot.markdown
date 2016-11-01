---
layout: post
title:  "Tools for directed or undirected graphing: dot"
date:   2016-10-31 13:49:14 +0800
categories: tools
---
Dot [main page][Dot] is a plain text graph description language. DOT graphs are typically files with the file extension gv or dot. Various programs can process DOT files. Some, such as OmniGraffle, dot, neato, twopi, circo, fdp, and sfdp, can read a DOT file and render it in graphical form. Others, such as gvpr, gc, acyclic, ccomps, sccmap, and tred, read DOT files and perform calculations on the represented graph. Finally, others, such as lefty, dotty, and grappa, provide an interactive interface. The GVedit tool combines a text editor with noninteractive image viewer. Graphviz [Graphviz home][Graphviz] is tool to parse the dot language file and output a vizualization picture.

The document introduces the dot details as well as some examples [demo gallery][demo-gallery]. Here I illustrated a simple example. The data comes from "ifstat" output, and I want to create a line graph according to that data. I can write command like:
{% highlight ruby linenos %}
dot -Tpng vssdot.txt -o vssdot.png
{% endhighlight %}
![Screenshot for dot output]({{site.url}}/assets/vssdot.png)

You can get [the test data here]({{site.url}}/assets/vssdot.txt)

[demo-gallery]: https://www.ocf.berkeley.edu/~eek/index.html/tiny_examples/thinktank/src/gv1.7c/doc/dotguide.pdf
[Dot]:      http://www.graphviz.org/doc/info/lang.html
[Graphviz]: http://www.graphviz.org/
