# An Observable Notebook

When you download an Observable notebook, you get a compressed folder of files. Several are basically the same for every notebook:

- **index.html** loads everything else; it imports the notebook definition, [Runtime](https://github.com/observablehq/runtime), [Standard Library](https://github.com/observablehq/stdlib), and [Inspector](https://github.com/observablehq/inspector). It does not hold the main content of your notebook; is customized only with your title.
- **package.json** includes basic metadata about your notebook to make it an npm-installable package.
- **runtime.js** includes a copy of the Runtime, Standard Library, and Inspector, so that your tarball is self-contained and runs without an Internet connection.
- **index.js** is the default entry point for the notebook package, and just points to your main notebook definition.
- **README.md**	includes a link to your notebook on observablehq.com and boilerplate information about how you can use the package.
- **inspector.css** styles the display of cells that return a JavaScript value (like an array or string), as opposed to a DOM node. It does not include the other styles (like fonts, colors, and tables) that you see when you render a cell on observablehq.com.

In this case we’ve downloaded the [D3 Line Chart](https://observablehq.com/@d3/line-chart@183); you can easily spot the files specific to that notebook because their names make no sense:

- **files/de259092d525c13bd1092…** is the one file attachment, a CSV of Apple stock prices (although the download does not retain the file extension).
- **35e84dc078248e17&#64;183.js**, which is the real heart of the whole thing, the contents of your notebook, your code, and which we will now go through in detail.

Your carefully handcrafted code is all here! But it takes a little practice to recognize it in its compiled state, where the magic of reactivity has to be implemented in plain JavaScript. The content of each cell is wrapped in a function, and each function is passed to the runtime as a cell definition. In this context, each cell is called a _variable_. They’re basically the same thing; what you see as a cell in the frontend is defined as a variable for the runtime.

     

Let’s start! Export the notebook module definition, which will be passed the `runtime` (which lets you define cells) and `observer` (which will be notified when things change) by `index.html`.

    export default function define(runtime, observer) {

      const main = runtime.module();

Observable’s [file attachments](https://observablehq.com/@observablehq/file-attachments) are implemented as a [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) from the filenames you use in your cells, like `"aapl.csv"`, to the downloaded file’s true URL; [`import.meta.url`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import.meta), when called inside this module, provides this file’s URL as the base for the [URL constructor](https://developer.mozilla.org/en-US/docs/Web/API/URL).

      const fileAttachments = new Map([
        [
          "aapl.csv",
          new URL(
            "./files/de259092d525c13bd10926eaf7add45b15f2771a8b39bc541a5bba1e0206add4880eb1d876be8df469328a85243b7d813a91feb8cc4966de582dc02e5f8609b7",
            import.meta.url
          ),
        ],
      ]);

The built-in FileAttachment function, which you call in your cells, is here redefined to use that Map.

      main.builtin(
        "FileAttachment",
        runtime.fileAttachments((name) => fileAttachments.get(name))
      );

Now we come to our first cell, the unnamed title cell of the notebook, which contains only Markdown text. To define it, call the `main` module’s `variable` method (to get a new undefined variable), and then that variable’s `define` method (to define it). 

      main

`.variable` is passed the `observer`, which will be notified if this cell changes.

        .variable(observer())

`.define` is passed two things: 

- An array of the names of other variables that this cell depends on, and
- A function that evaluates the content of the cell, passing in those other dependencies.

In this case, there’s only one dependency: `md`, the tagged template literal defined by the Standard Library to render Markdown as HTML.

        .define(
          ["md"],
          function (md) {

Here, finally, we have the familiar code you see in the first cell of this notebook on observablehq.com. The only difference is that it is being `return`ed, whereas on observablehq.com the return is implicit. 

            return md`
              # Line Chart

              This static time series line chart shows the daily close of Apple stock. Compare to a [log _y_-scale showing change](/@d3/change-line-chart), an [area chart](/@d3/area-chart), a [horizon chart](/@d3/horizon-chart-ii), a [candlestick chart](/@d3/candlestick-chart), and an [index chart](/@d3/index-chart). To inspect values, consider a [tooltip](/@d3/line-chart-with-tooltip).

              Data: [Yahoo Finance](https://finance.yahoo.com/lookup)`;

That concludes our first cell! It’s all downhill from here.

          });

Our second cell, a.k.a. our second variable, is different in one big way: it has a name, `chart`. That name is passed to `observer` so that it will notify other things that reference `chart` when this variable changes (or something???). Also, `define` is now passed _three_ things:

- New! The name of this variable
- Same: The other variables it depends on: `d3`, `width`, etc.
- Same: The function, passing in those dependencies.

      main
        .variable(observer("chart"))
        .define(
          "chart",
          ["d3", "width", "height", "xAxis", "yAxis", "data", "line"],
          function (d3, width, height, xAxis, yAxis, data, line) {

Now here we see the familiar code that you see in the second cell of this notebook on observablehq.com. On there you can see it is wrapped in curly braces, unlike the first cell, and here you can see how that works. When a cell is wrapped in curly braces, that’s the whole body of the function; when not, the compiler gives it a return statement. Both kinds of cell end up compiling to the same kind of function.

            const svg = d3.create("svg").attr("viewBox", [0, 0, width, height]);

            svg.append("g").call(xAxis);

            svg.append("g").call(yAxis);

            svg
              .append("path")
              .datum(data)
              .attr("fill", "none")
              .attr("stroke", "steelblue")
              .attr("stroke-width", 1.5)
              .attr("stroke-linejoin", "round")
              .attr("stroke-linecap", "round")
              .attr("d", line);

            return svg.node();
          }
        );

Our third cell works similarly.

      main
        .variable(observer("data"))
        .define(
          "data",
          ["d3", "FileAttachment"],

But this time, the function is `async`. That’s because we’re `await`ing the result of the `text` method of the FileAttachment, which is asynchronously fetched from the URL we defined above. Note that the Observable compiler has automatically made it an `async` function; on the website, the content of the cell begins with the `Object.assign…` part.

          async function (d3, FileAttachment) {
            return Object.assign(
              d3
                .csvParse(await FileAttachment("aapl.csv").text(), d3.autoType)
                .map(({ date, close }) => ({ date, value: close })),
              { y: "$ Close" }
            );
          }
        );

The rest of the variables are defined the same way. See if you can make sense of each of them. What do they depend on? Whenever one of those inputs changes, the variable’s function will be called with the new values. That’s how reactivity works.  

      main
        .variable(observer("line"))
        .define(
          "line",
          ["d3", "x", "y"],
          function (d3, x, y) {
            return d3
              .line()
              .defined((d) => !isNaN(d.value))
              .x((d) => x(d.date))
              .y((d) => y(d.value));
          }
        );

      main
        .variable(observer("x"))
        .define(
          "x",
          ["d3", "data", "margin", "width"],
          function (d3, data, margin, width) {
            return d3
              .scaleUtc()
              .domain(d3.extent(data, (d) => d.date))
              .range([margin.left, width - margin.right]);
          }
        );

      main
        .variable(observer("y"))
        .define(
          "y",
          ["d3", "data", "height", "margin"],
          function (d3, data, height, margin) {
            return d3
              .scaleLinear()
              .domain([0, d3.max(data, (d) => d.value)])
              .nice()
              .range([height - margin.bottom, margin.top]);
          }
        );

      main
        .variable(observer("xAxis"))
        .define(
          "xAxis",
          ["height", "margin", "d3", "x", "width"],
          function (height, margin, d3, x, width) {
            return (g) =>
              g.attr("transform", `translate(0,${height - margin.bottom})`).call(
                d3
                  .axisBottom(x)
                  .ticks(width / 80)
                  .tickSizeOuter(0)
              );
          }
        );

      main
        .variable(observer("yAxis"))
        .define(
          "yAxis",
          ["margin", "d3", "y", "data"],
          function (margin, d3, y, data) {
            return (g) =>
              g
                .attr("transform", `translate(${margin.left},0)`)
                .call(d3.axisLeft(y))
                .call((g) => g.select(".domain").remove())
                .call((g) =>
                  g
                    .select(".tick:last-of-type text")
                    .clone()
                    .attr("x", 3)
                    .attr("text-anchor", "start")
                    .attr("font-weight", "bold")
                    .text(data.y)
                );
          }
        );

One last twist: `margin` and `height` have names, but no dependencies, so you just pass the name and the function to `define`. To recap the overloading of `define`:

- `.define(Array, Function)` is equivalent to `.define(null, Array, Function)`.
- `.define(String, Function)` is equivalent to `.define(String, null, Function)`.

      main
        .variable(observer("margin"))
        .define(
          "margin",
          function () {
            return { top: 20, right: 30, bottom: 30, left: 40 };
          }
        );

      main
        .variable(observer("height"))
        .define(
          "height",
          function () {
            return 500;
          }
        );

I said earlier that cells and variables are basically the same thing. One exception is that there are built-in variables that aren’t represented by any cell. This last cell depends on one of them, `require`. We saw three other examples above: `FileAttachment`, `md`, and `width`.

      main
        .variable(observer("d3"))
        .define(
          "d3",
          ["require"],
          function (require) {
            return require("d3@5");
          }
        );

      return main;
    }

And that’s it! That’s how a compiled Observable notebook is defined for the Runtime.

For me, seeing the notebook in plain JavaScript clarified what the curly brace block syntax does; how reactive variables relate to the scope of a cell; and that asynchronous cells don’t look that scary. (One thing not shown here is a generator cell, with `yield` statements; maybe I should find a better example!) It also begins to demystify the Runtime, which enables very intricate interoperation. Finally, it helps understand how to copy and paste to port a notebook to a non-reactive setting (without the Runtime). If you pull out one of these functions, it’s plain JavaScript, and if you call it with the same arguments, you’ll get the same result.

Love, Toph