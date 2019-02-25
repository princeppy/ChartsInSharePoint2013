ChartsInSharePoint2013
======================

Building Charts in SharePoint 2013 using JavaScript and REST



# Creating a SharePoint Chart | JavaScript Charts

Published 11/12/2018 10:09 AM |  Updated 01/08/2019 10:28 AMIn a recent project, we were tasked with building a custom dashboard with different types of charts. The Chart web part previously available in SharePoint 2010 is no longer available in SharePoint 2013\. Excel Services and PowerPivot are good options but wouldn't work on this project due to the need to summarize data from multiple lists. As a result, we decided to create charts via JavaScript querying the data using REST.The completed solution contains a pie chart, a bar chart and a stacked bar chart. The final product generates the following dashboard:

![](https://www.insight.com/content/dam/insight-web/sitesections/knowledgebase/Cardinal/Images/BuildingCharts1.png)

In this post, we'll discuss:

* The structure of the data in the SharePoint lists
* How to retrieve the data via REST
* Selection of a JavaScript charting tool
* Building a pie chart in SharePoint 2013 using JavaScript and REST
* Building a bar chart in SharePoint 2013 using JavaScript and REST
* Building a stacked bar chart in SharePoint 2013 using JavaScript and REST

### Summary of the lists

As mentioned above, multiple lists contained the information we needed to summarize. Our goal was to make a pie chart detailing the number of engagements by type. For our example, we're going to aggregate the data from three different lists.The first list, called Empire Engagements, is a list of all engagements of the type Empire. It contains the title of the engagement, as well as the current status and leader of the engagement.

![](https://www.insight.com/content/dam/insight-web/sitesections/knowledgebase/Cardinal/Images/BuildingCharts2.png)

The second list, called Rebel Engagements, is a list of all engagements of the type Rebel. It also contains the title of the engagement, as well as the current status and leader of the engagement.
![](https://www.insight.com/content/dam/insight-web/sitesections/knowledgebase/Cardinal/Images/BuildingCharts3.png)
The final list is called Independent Engagements. Like the previous lists, this list contains the title, status and leader of the engagement, but it also contains an Engagement Type column. The Engagement Type column is a choice column that allows multiple selections.
![](https://www.insight.com/content/dam/insight-web/sitesections/knowledgebase/Cardinal/Images/BuildingCharts4.png)

### Retrieving the data

Two options exist for retrieving the data from SharePoint for our project: Representational State Transfer (REST) and the Client-Side Object Model (CSOM). While CSOM has been around since SharePoint 2010 and is currently better documented around the web, the preferred approach is to use REST as it's a standard that can be understood by multiple technologies. Certain tasks require CSOM as they can't be performed via REST, but the simple retrieval of items isn't one of those tasks.Using REST, we can easily query the data from the Empire Engagements list using a simple URL: http://sitename/\_api/web/lists/getByTitle('Empire%20Engagements')/items?$select=ID,Title,Status

This returns our data in an XML format that can be read in the browser or through our JavaScript code.
![](https://www.insight.com/content/dam/insight-web/sitesections/knowledgebase/Cardinal/Images/BuildingCharts5.png)
If we wanted to filter the data to only show items with an Active Status, we can change the URL to: http://sitename/\_api/web/lists/getByTitle('Empire%20Engagements')/items?$select=ID,Title,Status&$filter=Status eq 'Active'

To query the data via JavaScript, we can use the following code:

```
$.ajax({
url: _spPageContextInfo.webServerRelativeUrl +
"/_api/web/lists/getByTitle('Empire%20Engagements')/items?$select=ID",
    type: "GET",
    headers: {
        "accept": "application/json;odata=verbose",
    },
    success: function (data) {
        //Manipulate the data
    },
    error: function (err) {
        alert(JSON.stringify(err));
    }
});
```

Although querying the data isn't problematic using the pattern above with one list, it quickly becomes unmanageable when multiple sources of data are called.

```
$.ajax({
    url: _spPageContextInfo.webServerRelativeUrl + "/_api/web/lists/getByTitle('Empire%20Engagements')/items?$select=ID",
    type: "GET",
    headers: {
        "accept": "application/json;odata=verbose",
    },
    success: function (data) {
        $.ajax({
            url: _spPageContextInfo.webServerRelativeUrl + "/_api/web/lists/getByTitle('Rebel%20Engagements')/items?$select=ID",
            type: "GET",
            headers: {
                "accept": "application/json;odata=verbose",
            },
            success: function (data) {
                //Manipulate the data from both calls
            },
            error: function (err) {
                alert(JSON.stringify(err));
            }
        });
    },
    error: function (err) {
        alert(JSON.stringify(err));
    }
});
```

In cases where we need to gather information from three different lists, each additional $.ajax call would need to be made inside the previous call's success function. In order to prevent "spaghetti" code, we turn to the concept of promises or deferreds.

### Using jQuery promises

In order to implement promises in the JavaScript, we'll use jQuery Deferreds. Using the Deferred object, we can make multiple asynchronous calls to the SharePoint REST API at the same time, rather than waiting for each individual call to complete prior to starting the next call. Using deferreds also allows us to create a much cleaner structure in our code.

```
"use strict";

$.when(
    //Empire Engagements Query
    Engagements.RESTQuery.execute("Empire%20Engagements", "$select=ID"),
    //Rebel Engagements Query
    Engagements.RESTQuery.execute("Rebel%20Engagements", "$select=ID")
).done(
    function (engagements1, engagements2) {
        //Manipulate data
        //engagements1 contains results from Empire Engagements
        //engagements2 contains results from Rebel Engagements
    }
).fail(
    function (engagements1, engagements2) {
        //Capture error and display error message
        alert("An error has occurred");
    }
);

var Engagements = window.Engagements || {};

//Executing a REST query
Engagements.RESTQuery = function (listTitle, query) {
    var execute = function (listTitle, query) {
        var restUrl = _spPageContextInfo.webServerRelativeUrl +
            "/_api/web/lists/getByTitle('" + listTitle + "')/items";
        if (query != "") {
            restUrl = restUrl + "?" + query;
        }
        var deferred = $.ajax({
            url: restUrl,
            type: "GET",
            headers: {
                "accept": "application/json;odata=verbose",
                "X-RequestDigest": $("#__REQUESTDIGEST").val()
            }
        });

        return deferred.promise()
    };

    return {
        execute: execute
    }
}();
```

Additionally, this code contains some refactoring through the use of a generic function to retrieve data from lists via REST calls.

### JavaScript charting tool
There are several good JavaScript charting libraries available on the internet. We decided to use a library called Highcharts because of the look and good documentation. The website is [http://www.highcharts.com/](http://www.highcharts.com/). This sample uses the highcharts.js and exporting.js libraries.

### Loading the charts on the page
Although there are several ways to add the JavaScript and HTML needed for the charts, we took the simplistic approach of adding to a page via the Script Editor web part (under the Media and Content category). All of the common JavaScript files are added via one web part, and the HTML and JavaScript to create each chart is contained within its own web part. All of the JavaScript files are uploaded to the SiteAssets library.

### Building a pie chart in SharePoint 2013 using JavaScript and REST

Now we'll discuss the creation of the pie chart within our dashboard solution.

### Retrieving data
As mentioned above, we're using jQuery promises --- specifically the when, done, fail pattern --- to retrieve our data via REST. We're querying data from the Independent Engagements list, the Rebel Engagements list and the Empire Engagements list. The data is returned respectively into the engagements1, engagements2 and engagements3 variables.To build the pie chart, we need our data as an array of names and counts, such as the following:\[\["Rebel", 3\], \["Empire", 4\], \["Independent", 1\]\]We'll store these values in the countArray variable.Retrieving the data from the Rebel Engagements and Empire Engagements lists is easy. We simply need to count the number of results retrieved. The following lines of code take the counts and put them into the countArray.

```
  //Add count of Rebel Engagements list
  countArray.push(["Rebel", engagements2[0].d.results.length]);

  //Add count of Empire Engagements list
  countArray.push(["Empire", engagements3[0].d.results.length]);
```

Retrieving the data from the Independent Engagements lists is a little more difficult as we need to determine which category (or categories) the list item is assigned to. To temporarily hold the data, we place the data into a temporary data array called dataArray. This will simply be an array of all of the values --- we're not worried about counts at this point.

```
var results = engagements1[0].d.results;
for (var i = 0; i < results.length; i++) {
   var engagementType = results[i].Engagement_x0020_Type.results;
   for (var j = 0; j < engagementType.length; j++) {
       dataArray.push(engagementType[j]);
   }
}
 ```

With the code, we're looping through each of the list items in the for loop using the i counter variable. The value in the Engagement Type column can be retrieved through the following code:

    results[i].Engagement_x0020_Type 

This works perfectly for single choice or single value items, but in our case, we're using a choice column that allows multiple values. This information is returned as an array of data. In order to retrieve each selected choice, we must loop through the values. This is done in the for loop using the j counter variable.Now that we have all of the selected values, we need to determine a count for each of the engagement types and merge it with the countArray where we stored our counts from the Rebel Engagements list and the Empire Engagements list. The following code includes these counts into the countArray variable:

```
if (countArray == undefined) {
    countArray = [];
}

for (var i = 0; i < dataArray.length; i++) {
    var currValue = dataArray[i];
    var found = false;
    for (var j = 0; j < countArray.length; j++) {
        if (countArray[j][0] == currValue) {
            found = true;
            var newCount = countArray[j][1];
            countArray[j][1] = newCount + 1;
        }
    }
    if (!found) {
        countArray.push([currValue, 1]);
    }
}
if (countArray == undefined) {
    countArray = [];
}

for (var i = 0; i < dataArray.length; i++) {
    var currValue = dataArray[i];
    var found = false;
    for (var j = 0; j < countArray.length; j++) {
        if (countArray[j][0] == currValue) {
            found = true;
            var newCount = countArray[j][1];
            countArray[j][1] = newCount + 1;
        }
    }
    if (!found) {
        countArray.push([currValue, 1]);
    }
}
```

### Creating the chart
Now that we have the data in the format we need, we can create the chart. The chart will display in the engagementPieChart div on our page.

```
$('#engagementPieChart').highcharts({
   chart: {
        plotBackgroundColor: null,
        plotBorderWidth: null,
        plotShadow: false
   },
   credits: {
        enabled: false
   },
   title: {
        text: 'Engagements'
   },
   tooltip: {
        pointFormat: '{series.name}: <b>{point.y}</b>',
        percentageDecimals: 0
   },
   plotOptions: {
        pie: {
           allowPointSelect: true,
           cursor: 'pointer',
           dataLabels: {
               enabled: false
           },
           showInLegend: true
        }
   },
   series: [{
       type: 'pie',
       name: 'Engagements',
       data: countArray
    }]
});
```
### Building a bar chart in SharePoint 2013 using JavaScript and REST
Next, we'll discuss the creation of the bar chart within our dashboard solution.

### Retrieving data
We're using jQuery promises --- specifically the when, done, fail pattern --- to retrieve our data via REST. We're querying data from the Rebel Engagements list and the Empire Engagements list. The data is returned respectively into the engagements1 and engagements2 variables.To build the bar chart, we need our data separated into two arrays, with the category name and the corresponding count in the same position within each array, such as the following:

\["Luke Skywalker", "Darth Vader", "Yoda", "Princess Leia"\]

\[2, 4, 1, 1\]

To temporarily hold the data from the Rebel Engagements and Empire Engagements lists, we place the data into a temporary data array called dataArray. This will simply be an array of all of the values, not worrying about counts at this point.

```
//Get data from Rebel Engagements list
var results = engagements1[0].d.results;
for (var i = 0; i < results.length; i++) {
    var leader = results[i].Leader;
    dataArray.push(leader);
}
//Get data from Empire Engagements list
var results = engagements2[0].d.results;
for (var i = 0; i < results.length; i++) {
    var leader = results[i].Leader;
    dataArray.push(leader);
}
```

With the code, we're looping through each of the list items in the for loop using the i counter variable. The value in the Leader column can be retrieved through the following code: Keep in mind that the above code only works for single choice or single value items.Now that we have all of the selected values, we need to summarize the data into leaders and counts. The following code includes these counts into the countArray variable:

```
if (countArray == undefined) {
    countArray = [];
}

for (var i = 0; i < dataArray.length; i++) {
    var currValue = dataArray[i];
    var found = false;
    for (var j = 0; j < countArray.length; j++) {
        if (countArray[j][0] == currValue) {
            found = true;
            var newCount = countArray[j][1];
            countArray[j][1] = newCount + 1;
        }
    }
    if (!found) {
        countArray.push([currValue, 1]);
    }
}
if (countArray == undefined) {
    countArray = [];
}

for (var i = 0; i < dataArray.length; i++) {
    var currValue = dataArray[i];
    var found = false;
    for (var j = 0; j < countArray.length; j++) {
        if (countArray[j][0] == currValue) {
            found = true;
            var newCount = countArray[j][1];
            countArray[j][1] = newCount + 1;
        }
    }
    if (!found) {
        countArray.push([currValue, 1]);
    }
}
```

Now we need to separate the data into the two separate arrays with category name in one array and corresponding counts in the other, as mentioned above. This is done with the following code:

```
var seriesData = [];
var xCategories = [];
for (var i = 0; i < countArray.length; i++) {
    xCategories.push(countArray[i][0]);
    seriesData.push(countArray[i][1]);
}
```

The categories are now in the xCategories array, and the counts are in the seriesData array.

### Creating the chart
Now that we have the data in the format we need, we can create the chart. The chart will display in the engagementsByLeaderChart div on our page.

```
$('#engagementsByLeaderChart').highcharts({
  chart: {
      type: 'bar'
  },
  credits: {
      enabled: false
  },
  title: {
      text: chartTitle
  },
  xAxis: {
      categories: xCategories
  },
      yAxis: {
      min: 0,
      title: {
          text: yAxisTitle
      }
  },
  legend: {
      enabled: false
  },
  plotOptions: {
      bar: {
          dataLabels: {
             enabled: false
          }
      },
      series: {
          animation: false
      }
  },
  series: [{
      name: yAxisTitle,
      data: seriesData
  }]
});
```

### Building a stacked bar chart in SharePoint 2013 using JavaScript and REST
Finally, we'll discuss the creation of the stacked bar chart within our dashboard solution.

### Retrieving and manipulating data
As mentioned in the previous sections, we're using jQuery promises --- specifically the when, done, fail pattern --- to retrieve our data via REST. We're querying data from the Rebel Engagements list and the Empire Engagements list. The data is returned respectively into the engagements1 and engagements2 variables.To build the stacked bar chart, we need our data separated into a complex structure of objects and arrays, such as the following:

\["Luke Skywalker", "Princess Leia", "Yoda", "Darth Vader"\]  
\[{"Completed", \[1, 0, 0, 1\]},{"Active", \[1, 1, 1, 2\]},{"Pipeline", \[0, 0, 0, 1\]}\]The first array holds the leaders' names or categories, which will be displayed on the y-axis. The second array contains the status value (Pipeline, Active, Completed) within our chart, along with an array containing the values corresponding to the leader names.In order to hold the data we'll retrieve, we create a function that contains the leader name, status and count.

```
EngagementChartBuilder.StatusByLeader = function (name, status, count) {
    var leaderName = name,
        statusName = status,
        statusCount = count

    return {
        leaderName: name,
        statusName: statusName,
        statusCount: statusCount
    }
}
```

Then we retrieve the data and place it into an array, incrementing the counts when both the leader and status are matched.

```
//Get results from Rebel Engagements
var results = engagements1[0].d.results;
for (var i = 0; i < results.length; i++) {
    var found = false;
    for (var j = 0; j < data.length; j++) {
        if (data[j].leaderName == results[i].Leader &&
            data[j].statusName == results[i].Status) {
            data[j].statusCount = data[j].statusCount + 1;
            found = true;
        }
    }
    if (!found) {
        data.push(new EngagementChartBuilder.StatusByLeader(results[i].Leader,
                 results[i].Status, 1));
    }
}
//Get results from Empire Engagements Engagements
var results = engagements2[0].d.results;
for (var i = 0; i < results.length; i++) {
    var found = false;
    for (var j = 0; j < data.length; j++) {
        if (data[j].leaderName == results[i].Leader &&
            data[j].statusName == results[i].Status) {
            data[j].statusCount = data[j].statusCount + 1;
            found = true;
        }
    }
    if (!found) {
        data.push(new EngagementChartBuilder.StatusByLeader(results[i].Leader,
                 results[i].Status, 1));
    }
}
```
Now we have the data in the following format:

\[{"Luke Skywalker", "Active", 1}, {"Luke Skywalker", "Completed", 1},  
{"Darth Vader", "Active", 2}, {"Darth Vader", "Pipeline", 1}, ...\]Next, we separate out the categories (i.e., the leader names) into the xCategories array and the status values into the xStatus array:

```
//Get Categories (Leader Name)
for (i = 0; i < data.length; i++) {
    cat = data[i].leaderName;
    if (xCategories.indexOf(cat) === -1) {
        xCategories[xCategories.length] = cat;
    }
}
//Get Status values
for (i = 0; i < data.length; i++) {
    stat = data[i].statusName;
    if (xStatus.indexOf(stat) === -1) {
        xStatus[xStatus.length] = stat;
    }
}
```

Then, in order to have a place to update the counts, we create the complex structure of objects and arrays needed for the stacked bar chart. As a start, we're creating all of the counts as zero so we just have a placeholder for each value.

```
//Create initial series data with 0 values
for (i = 0; i < xStatus.length; i++) {
    var dataArray = [];
    for (j = 0; j < xCategories.length; j++) {
        dataArray.push(0);
    }
    seriesData.push({ name: xStatus[i], data: dataArray });
}
```
    
As the final step of manipulating the data, we loop through all possible combinations of leaders and status values while counting up matching records in our data retrieved from the REST calls. This data goes into our seriesData array.

```
//Cycle through data to assign counts to the proper location in the series data
for (i = 0; i < data.length; i++) {
    var leaderIndex = xCategories.indexOf(data[i].leaderName);
    for (j = 0; j < seriesData.length; j++){
        if (seriesData[j].name == data[i].statusName){
            seriesData[j].data[leaderIndex] = data[i].statusCount;
            break;
        }
    }
}
```

#### Creating the chart
Now that we have the data in the format we need, we can create the chart. The chart will display in the engagementsByStatusChart div on our page.

```
$('#engagementsByStatusChart').highcharts({
    chart: {
        type: 'bar'
    },
    colors: ['#339933', '#903b3b', '#583596', '#447088', '#5a7952', '#838843'],
    credits: {
        enabled: false
    },
    title: {
        text: 'Engagements by Status'
    },
    xAxis: {
        categories: xCategories
    },
    yAxis: {
        min: 0,
        title: {
            text: 'Total Engagements'
        }
    },
    legend: {
        backgroundColor: '#FFFFFF',
        reversed: true
    },
    plotOptions: {
        series: {
            animation: false,
            stacking: 'normal'
        }
    },
    series: seriesData
});
```
    
For more details on the parameters and options used to create the chart, look at the Highcharts API reference at [http://api.highcharts.com/highcharts](http://api.highcharts.com/highcharts).

### Conclusion
This article has described how to build a dashboard in SharePoint 2013 using JavaScript and REST. We've included a pie chart, a bar chart and a stacked bar chart. The full source is available at Github: [https://github.com/CardinalNow/ChartsInSharePoint2013](https://github.com/CardinalNow/ChartsInSharePoint2013).

### [Explore more Digital Innovation insights →](https://www.insight.com/en_US/solve/digital-innovation/insights.html)

This article originally appeared on May 7, 2013\.

[Source](https://help.insight.com/app/answers/detail/a_id/128)
